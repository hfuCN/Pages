---
title: 内核开发之消息队列
layout: post
guid: urn:uuid:7f79827f-da0b-4d6c-b0f0-97dcfeff7ff1
tags:
  - linux
  - kernel
  - c
---

消息队列是线程交互的一种方法，任务可以通过消息队列来实现数据的沟通和交换。在嵌入式系统上，这可以说这是用的最多的一种方法。通过消息队列，无论是发送者，还是接受者都可以循环地处理各种消息。而我们知道，存储消息最好的方式就是循环队列，如果消息已满，那么发送者可以把自己pend到等待队列上；而如果此时没有消息，那么接受者也可以把自己pend到等待队列上。当然实现消息队列的方法很多，甚至用户可以自己利用互斥量和信号量来实现，而嵌入式系统常常会默认提供这样的功能函数，我想主要的目的还是为了方便用户，让他们可以更多地从业务的角度来看问题，而不是把重点关注在这些底层的细节上面。

首先，我们还是看看rawos上面关于消息队列的数据结构是怎么定义的，

```
typedef struct RAW_MSG_Q {    
  	
  		RAW_VOID         **queue_start;       /* Pointer to start of queue data  */
  		RAW_VOID         **queue_end;         /* Pointer to end   of queue data  */
  		RAW_VOID         **write;             /* Pointer to where next message will be inserted in the Q  */
  		RAW_VOID         **read;              /* Pointer to where next message will be extracted from the Q  */
  		RAW_U32          size;                /* Size of queue (maximum number of entries) */
  		RAW_U32          current_numbers;               /* Current number of entries in the queue */
  		RAW_U16          blocked_send_task_numbers;     /*number of blocked send task numbers */
  		RAW_U16          blocked_receive_task_numbers;  /*number of blocked send task numbers */
} RAW_MSG_Q;
  
typedef struct RAW_QUEUE
{ 
	  	RAW_COMMON_BLOCK_OBJECT       common_block_obj;
  		RAW_MSG_Q                     msg_q;
  	
} RAW_QUEUE;

```

上面的代码中有两段数据结构，第一段主要表示循环队列的内容，其中包括了队列首地址、队列末尾地址、当前队列读取地址、当前队列插入地址、队列大小、消息个数、阻塞的发送线程数据、阻塞的接受线程数目。而第二段数据结构就比较简单，它把通用等待结构和循环队列合在了一起，共同构成了消息队列的数据结构。
 
根据我们以前的经验，互斥同步数据结构的操作都会分成几个部分，当然消息队列也不例外，也会分成初始化、发送消息、接受消息、清除消息、删除消息队列等几种操作函数。当然，消息队列还是增加了一个新的选项，那就是插入消息的时候可以插入在队列的前方，还是插入在队列的尾部，这在某种程度上决定了消息的优先级。说到这，我们还是看看消息队列是怎么初始化的，

```
RAW_U16 raw_queue_create(RAW_QUEUE *p_q, RAW_U8 *p_name, RAW_VOID **msg_start, RAW_U32 number)
{
  		#if (RAW_QUEUE_FUNCTION_CHECK > 0)
  		
  		if (p_q == 0) {
  				return RAW_NULL_OBJECT;
  		}
  		
   		if ( msg_start == 0) {
  				return RAW_NULL_POINTER;
  		}
  	
  		if (number == 0) {
  				return RAW_ZERO_NUMBER;
  		}
  	
  		#endif
  
  		list_init(&p_q->common_block_obj.block_list);
  	                             
  		p_q->common_block_obj.name              = p_name;   
  		p_q->common_block_obj.block_way         = 0;
  		p_q->msg_q.queue_start                  = msg_start;            /* Initialize the queue */
 	 	p_q->msg_q.queue_end                    = &msg_start[number];
  		p_q->msg_q.write                        = msg_start;
 	 	p_q->msg_q.read                         = msg_start;
  		p_q->msg_q.size                         = number;
  		p_q->msg_q.current_numbers              = 0;
  		p_q->msg_q.blocked_send_task_numbers    = 0;
  		p_q->msg_q.blocked_receive_task_numbers = 0;
 	 	return RAW_SUCCESS;
}
 
```

虽然相比较之前的互斥函数，消息队列的初始化内容好像多一些。但是大家如果对循环队列的知识比较了解的话，其实也不是很复杂的。我们看到，函数除了对通用阻塞结构进行初始化之外，就是对这些循环队列进行初始化。接着，我们就可以看看消息发送函数是怎么样的，

```
static RAW_U16 internal_msg_post(RAW_QUEUE *p_q, RAW_VOID *p_void,  RAW_U8 opt_send_method, RAW_U8 opt_wake_all, RAW_U32 wait_option)             
{
  		RAW_U16 error_status;
  		LIST *block_list_head;
  		RAW_U8 block_way;
  	
   		RAW_SR_ALLOC();
  
  		#if (RAW_QUEUE_FUNCTION_CHECK > 0)
  
  		if (raw_int_nesting) {
  				if (wait_option != RAW_NO_WAIT) {
  						return RAW_NOT_CALLED_BY_ISR;
  				}
  		}
  
  		if (p_q == 0) {
  				return RAW_NULL_OBJECT;
  		}
  	
  		if (p_void == 0) {
  				return RAW_NULL_POINTER;
  		}
  	
  		#endif
  
  		block_list_head = &p_q->common_block_obj.block_list;
  	
  		RAW_CRITICAL_ENTER();
  
  		/*queue is full condition, there should be no received task blocked on queue object!*/
  		if (p_q->msg_q.current_numbers >= p_q->msg_q.size) {  
  				if (wait_option  == RAW_NO_WAIT) {
  						RAW_CRITICAL_EXIT();
  						return RAW_MSG_MAX;
  				} else {
  			
  						/*system is locked so task can not be blocked just return immediately*/
  						if (raw_sched_lock) {   
  								RAW_CRITICAL_EXIT();
  								return RAW_SCHED_DISABLE;    
  						}
  						/*queue is full and  SEND_TO_FRONT  method is not allowd*/
  						if (opt_send_method == SEND_TO_FRONT) {
  								RAW_CRITICAL_EXIT();
  								return RAW_QUEUE_FULL_OPT_ERROR;  
  						}
  
  						p_q->msg_q.blocked_send_task_numbers++;
  						raw_task_active->msg = p_void;
  						block_way = p_q->common_block_obj.block_way;
  						p_q->common_block_obj.block_way = RAW_BLOCKED_WAY_FIFO;
  						/*there should be no blocked received task beacuse msg exits*/
  						raw_pend_object(&p_q->common_block_obj, raw_task_active, wait_option);
  						p_q->common_block_obj.block_way = block_way;
  			
  						RAW_CRITICAL_EXIT();
  
  						raw_sched(); 
  			
  						error_status = block_state_post_process(raw_task_active, 0);
  
  						return error_status;	
  				}
  
  		}
  
  		/*Queue is not full here, there should be no blocked send task*/	
  		/*If there is no blocked receive task*/
  		if (is_list_empty(block_list_head)) {        
  				p_q->msg_q.current_numbers++;     /* Update the nbr of entries in the queue */
  		
  				if (opt_send_method == SEND_TO_END)  {
  						*p_q->msg_q.write++ = p_void;                              
  
  						if (p_q->msg_q.write == p_q->msg_q.queue_end) {   
  								p_q->msg_q.write = p_q->msg_q.queue_start;
  				
  						}   
  
  				} else {
  						if (p_q->msg_q.read == p_q->msg_q.queue_start) {              
  	        					p_q->msg_q.read = p_q->msg_q.queue_end;
  	    				}
  			
  						p_q->msg_q.read--;
  						*p_q->msg_q.read = p_void;    /* Insert message into queue */
  			
  				}
  		
  				RAW_CRITICAL_EXIT();
  		
  				return RAW_SUCCESS;
  		}
  
  		/*wake all the task blocked on this queue*/
  		if (opt_wake_all) {
  				while (!is_list_empty(block_list_head)) {
  						wake_send_msg(list_entry(block_list_head->next, RAW_TASK_OBJ, task_list),  p_void);	
  				}
  		
  				p_q->msg_q.blocked_receive_task_numbers = 0;
  		}
  	
  		/*wake hignhest priority task blocked on this queue and send msg to it*/
  		else {
  				wake_send_msg(list_entry(block_list_head->next, RAW_TASK_OBJ, task_list),  p_void);	
  				p_q->msg_q.blocked_receive_task_numbers--;
  		}
  	
  		RAW_CRITICAL_EXIT();
  
  		raw_sched();    
  		return RAW_SUCCESS;
}
 
```

这里消息发送函数稍显冗长，这主要是因为消息发送的情况比较复杂，方方面面考虑的情况比较多。但是整个函数处理的逻辑还是比较清晰的，只要有耐心，慢慢读下去还是没有什么问题。这里不妨和大家一起看一下消息发送函数是怎么实现的，

     （1）检验参数合法性，注意在中断下调用这个函数时，必须是RAW_NO_WAIT的选项，中断毕竟是不好调度的；
     （2） 处理消息已满的情况，
             a）如果线程不想等待，函数返回；
             b）如果禁止调度，函数返回；
             c）消息存储到线程的msg里面，线程把自己pend到等待队列中；
             d）调用系统调度函数，等待再次被调度的机会，函数返回。
     （3）当前消息未满，但是当前没有等待队列，那么根据要求把消息压入循环队列，函数返回；
     （4）当前消息未满，且存在等待队列，说明此时已经没有消息可读，	
             a）如果需要唤醒所有的等待线程，那么唤醒所有的线程，等待线程总数置为0；
             b）如果只是唤起某一个线程，那么唤醒第一个等待线程，等待线程总数自减；
     （5）调用系统调度函数，防止有高优先级的线程加入调度队列；
     （6）线程再次得到运行的机会，函数返回。
 
看到上面的代码，我们发现只要梳理好了代码的逻辑，其实消息发送函数也是比较好理解的。当然，有消息的发送，就必然会存在消息的接受了。此时肯定也会出现没有消息、有消息两种情况了。

```
RAW_U16 raw_queue_receive (RAW_QUEUE *p_q, RAW_U32 wait_option, RAW_VOID  **msg)
{
  		RAW_VOID *pmsg;
  		RAW_U16 result;
  		LIST *block_list_head;
  		RAW_TASK_OBJ *blocked_send_task;
  	
  		RAW_SR_ALLOC();
  
  		#if (RAW_QUEUE_FUNCTION_CHECK > 0)
  
  		if (raw_int_nesting) {
  				return RAW_NOT_CALLED_BY_ISR;
  		
  		}
  
  		if (p_q == 0) {
  				return RAW_NULL_OBJECT;
  		}
  	
  		if (msg == 0) {
  				return RAW_NULL_POINTER;
  		}
  	
  		#endif
  
  		block_list_head = &p_q->common_block_obj.block_list;
  	
  		RAW_CRITICAL_ENTER();
  
  	
    	/*if queue has msgs, just receive it*/
  		if (p_q->msg_q.current_numbers) { 
  				pmsg = *p_q->msg_q.read++;                    
  		
  				if (p_q->msg_q.read == p_q->msg_q.queue_end) {         
  						p_q->msg_q.read = p_q->msg_q.queue_start;
  				}
  
  				*msg = pmsg;
  
  				/*if there are  blocked_send_tasks, just reload the task msg to end*/
  				if (p_q->msg_q.blocked_send_task_numbers) {
  						blocked_send_task = list_entry(block_list_head->next, RAW_TASK_OBJ, task_list);
  			
  						p_q->msg_q.blocked_send_task_numbers--;
  			
  						*p_q->msg_q.write++ = blocked_send_task->msg;                              
  
  						if (p_q->msg_q.write == p_q->msg_q.queue_end) {   
  								p_q->msg_q.write = p_q->msg_q.queue_start;		
  						}   
  			
  						raw_wake_object(blocked_send_task);
  						RAW_CRITICAL_EXIT();
  			
  						raw_sched();  
  						return RAW_SUCCESS;
  				}
  
  				p_q->msg_q.current_numbers--;  
  		
  				RAW_CRITICAL_EXIT();
  		
  				return RAW_SUCCESS;                         
  		}
  
  		if (wait_option == RAW_NO_WAIT) {    /* Caller wants to block if not available? */
  				*msg = (RAW_VOID *)0;
  				RAW_CRITICAL_EXIT();
  				return RAW_NO_PEND_WAIT;
  		} 
  
  		if (raw_sched_lock) {   
  				RAW_CRITICAL_EXIT();	
  				return RAW_SCHED_DISABLE;    
  		}
  
  		raw_pend_object(&p_q->common_block_obj, raw_task_active, wait_option);
  		p_q->msg_q.blocked_receive_task_numbers++;
  	
  		RAW_CRITICAL_EXIT();
  	
  		raw_sched();                                             
  
  		RAW_CRITICAL_ENTER();
  	
  		*msg = (RAW_VOID *)0;
  		result = block_state_post_process(raw_task_active, msg);
  	
  		RAW_CRITICAL_EXIT();  
  
  		return result; 	
  }
 
```

和发送消息函数相比，接受消息的操作还是要少一些，不要紧，大家一起来看一下实现逻辑，

     （1）判断参数合法性；
     （2）如果当前存在消息，
             a）读取循环队列中的消息；
             b）判断当前是否存在等待线程，因为之前有可能存在没有压入队列的消息，那么此时压入消息，唤醒该线程即可，调用系统调度函数返回；
             c）没有等待线程，消息总数自减，函数返回。
     （3）当前没有消息，
             a）线程不愿等待，函数返回；
             b）系统禁止调度，函数返回；
             c）线程将自己pend到等待队列中；
             d）调用系统调度函数，切换到其他线程继续运行；
             e）线程再次获得运行的机会，从thread结构中获取返回结果，函数返回。
 
 和发送消息、接受消息比较起来，清除消息和删除消息的处理就比较简单了。为了说明问题，我们不妨放在一起讨论一下，
 
 ```
RAW_U16 raw_queue_flush(RAW_QUEUE  *p_q)
{
  		LIST *block_list_head;
  
  		RAW_SR_ALLOC();
  	
  		RAW_TASK_OBJ *block_task;
  	
  		#if (RAW_QUEUE_FUNCTION_CHECK > 0)
  
  		if (p_q == 0) {	
  				return RAW_NULL_OBJECT;
  		}
  	
  		#endif
  	
  		block_list_head = &p_q->common_block_obj.block_list;
  	
  		RAW_CRITICAL_ENTER();
  
  		/*if queue is full and task is blocked on this queue, then wake all the task*/
  		if (p_q->msg_q.current_numbers >= p_q->msg_q.size) {
  				while (!is_list_empty(block_list_head)) {
  						block_task = list_entry(block_list_head->next, RAW_TASK_OBJ, task_list);
  						raw_wake_object(block_task);
  						block_task->block_status = RAW_B_ABORT;
  				}
  
  				p_q->msg_q.blocked_send_task_numbers = 0;
  		}
  	
  		RAW_CRITICAL_EXIT(); 
  	
  		raw_sched();
  	
  		return RAW_SUCCESS;
}
#endif
  
  
#if (CONFIG_RAW_QUEUE_DELETE > 0)
RAW_U16 raw_queue_delete(RAW_QUEUE *p_q)
{
  		LIST  *block_list_head;
  	
  		RAW_SR_ALLOC();
  
  		#if (RAW_QUEUE_FUNCTION_CHECK > 0)
  
 	 	if (p_q == 0) {	
  				return RAW_NULL_OBJECT;
  		}
  	
  		#endif
  
  		block_list_head = &p_q->common_block_obj.block_list;
  	
  		RAW_CRITICAL_ENTER();
  
  		/*All task blocked on this queue is waken up*/
  		while (!is_list_empty(block_list_head))  {
  				delete_pend_obj(list_entry(block_list_head->next, RAW_TASK_OBJ, task_list));	
  		}                             
  	
  		RAW_CRITICAL_EXIT();
  
  		raw_sched(); 
  	
  		return RAW_SUCCESS;
  	
}
#endif
 
 ``` 
 
  从代码据结构上也看得出来，两个函数的处理逻辑十分相像，所以可以放在一起研究一下，
  
     （1）判断参数合法性；
     （2）唤醒等待线程，这里消息清除函数唤醒的是发送线程，而消息删除函数唤醒的所有线程；
     （3）调用系统调度函数，切换到其他线程继续运行；
     （4）当前线程再次获得运行的机会，函数返回，一切ok搞定。
