<?php
namespace WeDevs\PM_Pro\Integrations\Controllers;

use WP_REST_Request;
use WeDevs\PM\Task\Controllers\Task_Controller;
use WeDevs\PM\Comment\Controllers\Comment_Controller;
use WeDevs\PM\Task\Models\Task;
use WeDevs\PM\Comment\Models\Comment;
use WeDevs\PM_Pro\Integrations\Models\Integrations ;
use WeDevs\PM\Activity\Models\Activity ;


class Integrations_Controller {

    public function index( WP_REST_Request $request ) {

        $request_header = $request->get_header('X-GitHub-Delivery');
        if(!empty($request_header) && isset($request_header)){
            $request_body = json_decode($request->get_body()) ;
            if(property_exists($request_body, 'issue')) {
                $task_controller = new Task_Controller();
                $comment_controller = new Comment_Controller();
                if($request_body->action == 'opened') {
                    $data = [];
                    $project_id = $request->get_param('project_id'); // [project_id] => 39
                    $data['assignees'] = array(0); // [assignees] => Array([0] => 0)
                    $data['title'] = $request_body->issue->title; // [title] => hahaha
                    $data['description'] = $request_body->issue->body; // [description] => hahaha
                    $data['estimation'] = 0; //[estimation] => 0
                    $data['board_id'] = ''; // [board_id] => 39
                    $data['list_id'] = $project_id; // [list_id] => 39
                    $data['privacy'] = false; // [privacy] => false
                    $data['is_admin'] = 1; // [is_admin] => 1
                    $request->set_param('action', '');
                    $request->set_param('issue', '');
                    $request->set_param('repository', '');
                    $request->set_param('sender', '');
                    $request->set_query_params($data);
                    $task_contoller_response =  $task_controller->store($request);

                    $activity_data = $task_contoller_response['activity']['data'] ;
                    $activity_data['meta']['int_source'] = 'github' ;
                    Activity::where('id',$activity_data['id'] )
                        ->update(['meta' => serialize($activity_data['meta'])]);

                    $result = [
                        'action' => 	$request_body->action,
                        'project_id' => 	$task_contoller_response['data']['project_id'],
                        'primary_key' =>  	$request_body->issue->id,
                        'foreign_key' => $task_contoller_response['data']['id'],
                        'type' => 'issues',
                        'source' => 'Github',
                    ];
                    $integrations = Integrations::create( $result );
                    return $integrations ;
                }
                if($request_body->action == 'deleted'){
                    if($request->get_header('X-GitHub-Event') == 'issues') {
                        $task = Integrations::where('primary_key', $request_body->issue->id)
                            ->where('type','issues')
                            ->first();
                        $request->set_param('project_id', $task->project_id);
                        $request->set_param('task_id', $task->foreign_key);
                        $task_contoller_response = $task_controller->destroy($request);

                        $activity_data = $task_contoller_response['activity']['data'] ;
                        $activity_data['meta']['int_source'] = 'github' ;
                        Activity::where('id',$activity_data['id'] )
                            ->update(['meta' => serialize($activity_data['meta'])]);

                    }
                    if($request->get_header('X-GitHub-Event') == 'issue_comment') {

                        $comment = Integrations::where( 'primary_key', $request_body->comment->id )
                            ->where('type','issues_comments')
                            ->first();
                        $request->set_param('comment_id',$comment->foreign_key);
                        $comment_controller_response = $comment_controller->destroy($request);
                        $activity_data = $comment_controller_response['activity']['data'] ;
                        $activity_data['meta']['int_source'] = 'github' ;
                        Activity::where('id',$activity_data['id'] )
                            ->update(['meta' => serialize($activity_data['meta'])]);
                    }
                }
                if($request_body->action == 'closed'){
                    $task = Integrations::where( 'primary_key', $request_body->issue->id )
                        ->where('type','issues')
                        ->first();
                    $request->set_param('task_id',$task->foreign_key);
                    $request->set_param('status', 1);
                    $task_controller_response = $task_controller->change_status($request);
                    $activity_data = $task_controller_response['activity']['data'] ;
                    $activity_data['meta']['int_source'] = 'github' ;

                    Activity::where('id',$activity_data['id'] )
                        ->update(['meta' => serialize($activity_data['meta'])]);
                }
                if($request_body->action == 'reopened'){
                    $task = Integrations::where( 'primary_key', $request_body->issue->id )
                        ->where('type','issues')
                        ->first();
                    $request->set_param('task_id',$task->foreign_key);
                    $request->set_param('status', 0);
                    $task_controller_response = $task_controller->change_status($request);
                    $activity_data = $task_controller_response['activity']['data'] ;
                    $activity_data['meta']['int_source'] = 'github' ;

                    Activity::where('id',$activity_data['id'] )
                        ->update(['meta' => serialize($activity_data['meta'])]);
                }
                if($request_body->action == 'created'){
                    $task = Integrations::where( 'primary_key', $request_body->issue->id )
                        ->where('type','issues')
                        ->first();

                    $request->set_param('project_id',$task->project_id);
                    $request->set_param('commentable_id',$task->foreign_key);
                    $request->set_param('content','<p>'.$request_body->comment->body.'</p>');
                    $request->set_param('commentable_type','task');
                    $comment_contoller_response = $comment_controller->store($request);

                    $activity_data = $comment_contoller_response['activity']['data'] ;
                    $activity_data['meta']['int_source'] = 'github' ;

                    Activity::where('id',$activity_data['id'] )
                        ->update(['meta' => serialize($activity_data['meta'])]);


                    $result = [
                        'action' => 	$request_body->action,
                        'project_id' => 	$task->project_id,
                        'primary_key' =>  	$request_body->comment->id,
                        'foreign_key' => $comment_contoller_response['data']['id'],
                        'type' => 'issues_comments',
                        'source' => 'Github',
                    ];
                    $integrations = Integrations::create( $result );
                    return $integrations ;
                }
                if($request_body->action == 'edited'){
                    if($request->get_header('X-GitHub-Event') == 'issues'){
                        $task = Integrations::where( 'primary_key', $request_body->issue->id )
                            ->where('type','issues')
                            ->first();
                        $request->set_param('project_id',$task->project_id);
                        $request->set_param('task_id',$task->foreign_key);
                        $request->set_param('title',$request_body->issue->title);
                        $request->set_param('description',$request_body->issue->body);
                        $task_controller_response = $task_controller->update($request);

                        $activity_data = $task_controller_response['activity']['data'] ;
                        $activity_data['meta']['int_source'] = 'github' ;

                        Activity::where('id',$activity_data['id'] )
                            ->update(['meta' => serialize($activity_data['meta'])]);


                    }
                    if($request->get_header('X-GitHub-Event') == 'issue_comment'){

                        $integrations = Integrations::where( 'primary_key', $request_body->comment->id )
                            ->where('type','issues_comments')
                            ->first();
                        $comment = Comment::where( 'id', $integrations->foreign_key )->first();
                        $request->set_param('project_id',$comment->project_id);
                        $request->set_param('comment_id',$comment->id);
                        $request->set_param('content',$request_body->comment->body);
                        $request->set_param('commentable_id',$comment->commentable_id);
                        $request->set_param('commentable_type','task');
                        $comment_controller_response = $comment_controller->update($request);

                        $activity_data = $comment_controller_response['activity']['data'] ;
                        $activity_data['meta']['int_source'] = 'github' ;

                        Activity::where('id',$activity_data['id'] )
                            ->update(['meta' => serialize($activity_data['meta'])]);
                    }
                }
            }
        }
    }
}
