the firmware of NVDLA is in charge of the operation of each hardware engine and core scheduling task.
In headed implementation, it is to be located on the nvdla microprocessor, 
In headless implementation, it is included as a part of the kernel. 

cf) shift+ctrl+f searches the entire folder directory in visual studios code 

function descriptions partly from 
http://nvdla.org/sw/runtime_environment.html#c.dla_register_driver

---
Core Engine Interface
main KMD functionalities, some are called from the portability layer (??Should be the other way around), 
(?)while others are called from runtime(?)

"engine.c" 
some common sense: 'engine' refers to a core component of a complex system. Classically, it is packaged as a library, that provides functionality to the software that loads it. 
Engines do not in and of themselves generally have a standlone user interfaces, or 'main', as they are not applications. 
Thus a distinguishable characteristics of an engine is its presenation as an applications

"engine_data.c"
dla_register_driver (Core Engine Interface - register driver with firmware during probe)
    Called once during boot to initialize DLA engine scheduler and register driver with firmware
    on boot time:
    1. pass pointer to driver context, which is passed as a param on each firmware call to portability layer functions
    2. update pointer to engine context, which is passed as a param on each function call to firmware 
    ~ "engine_context, the engine-specific data is received via this function"

"scheduler.c"
dla_execute_task (Core Engine Interface - Driver submits task info for execution)
    initilize sub-engines and start task execution. Events received from hardware triggers furthur tasks. 
    cf) uses dla_task_descriptor structure, to include all info on task.
    **MUST DO MORE INDEPTH ANALYSIS OF THIS FN 
    cf) ROI - Region of Interest

dla_process_events (Core Engine Interface - Bottom half caller to process events after interrupt)
    while the interrupt handler(from engine_isr.c) just records and not process events, 
    Portability layer must also call this fn after interrupt handler is done. 
    shares lock with dla_isr_handler. 

dla_clear_test (Core Engine Interface - clean task and engine state)
    reset engine scheduler, called by portability layer after task completion. 

"engine_isr.c"
dla_isr_handler (Core Engine Interface - Interrupt received from hardware)
    ISR - Interrupt Service Routine
    (usage in port/linux/nvdla_core_callbacks.c/nvdla_engine_isr - the portability layer)
    Called when DLA interrupt is received; implemented on the KMD-level. Thus a OS-specific portability layer handler must also be implemented, and call this function from that OS_specific handler.
    Protected by lock to prevent handling when firmware is programming layers (dla_process_events)

---
Portability Layer 
OS-dependent functionalities, located on the port/linux directory. 
Naturally calls some sort of Linux-dependent fns. 

"/port/linux/nvdla_core_callbacks.c"
dla_reg_read & dla_reg_write
    address is calculated as base(derived from driver context) + addr(input param)
    calls 'readl' & 'writel' commands - from linux io.h to access arm register on linux


dla_data_read
    read data from buffers passed by UMD in local memory. 
    defined on nvdla_core_callbacks.c , access to dma_buf (APIs on dma-buf linux kernel module.)
    the fn used by all dla data read 
    (usage of DMA-buffer for peripheral devices) 

dla_data_write
    write data from local buffer to UMD buffer (dma-buf)
    usage on schedular.c -> op_completion

and now fns that use param 'destination' 
    DESTINATION_PROCESSOR : Memory will be access by processor running firmware
    DESTINATION_DMA : Memory will be accessed by NVDLA DMA engines

dla_get_dma_address (nvdla_core_callbacks.c)
    a wrapper fn to call the corresponding read_address fn by each 'destination'
  
    dla_read_dma_address (DESTINATION_DMA)
    Read DMA address from address list 
    Used by function block programming operations to read address for DMA engines(that each hardware engines have). 
    calls nvdla_gem_dma_addr fn (/port/linux/nvdla_gem.c), which returns a memory handle
    nvdla_gem_dma_addr goes looks up GEM objects
   

    But what is this 'DMA address' used for? 
        DMA in the NVDLA scope refers to that data transaction from DRAM/SRAM to Buffer of each hardware engine
        Thus the management of shared memory region. 

    dla_read_cpu_address (DESITNATION_PROCESSOR)
        Read CPU accessible address 
        Used by engine schedular to read data from 'buffer shared by UMD' into 'local buffer' for consumption



---
Function Block Programming 
so when are the individual engine firmware called? Schedular classifies task, then pass on to somewhere? 

"conv.c" &  other engine firmwares. 


---
Trace analysis of LeNet 

fns that include "Enter: __func__ "
    1) dla_op_enabled(dla_processor_group)
        (Enter: )
        dla_update_consumer ('update dependency graph for this task')
        (Exit: )
    
    2) dla_op_programmed(dla_processor, dla_processor_group, rdma_id)
        (Enter: )
        group->pending = 0
        dla_update_consumer ('update dependency graph for this task')
        (Exit: )
    
    3) dla_read_config (dla_task, dla_processor, dla_processor_group)
        (Enter: )
        dla_get_engine
        dla_data_read
        (Exit: )
    
    4) dla_prepare_operation(dla_processor, dla_common_op_desc, roi_index, group_number)
        (Enter: )
        utils_get_free_group
        debug information trace ("processor: processor_name, group_id, rdma_group_id, available")
        ('update operation descriptor')
        dla_read_config
        group->pending = 1
        (Exit: )

    5) dla_program_operation(dla_processor, dla_processor_group)
        (Enter: )
        program trace ('program, operation index, ROI, group_id')
        group->programming = 1
        program(group)
        ('prefetch consumers')
            dla_get_op_desc
            **Here, fused_parent is also involved "Invalid fused op type warning"
        dla_op_programmed (see number 2)
        (Exit: )
    
    6) dla_enable_operation(dla_processor, dla_common_op_desc)
        (Enter: )
        ('find out if operation if already programmed' , goto enable_op)
            thus an operation must be 'programmed' before 'enabled'
        enable_op flow:
        (if event is triggered as a part of same groups programming, skip enable, as it will be programmed after the current one is complete )
            ('Enable processor_name, operation index, ROI')
            enable(group)
            dla_op_enabled(group)
        (Exit: )

    7) dla_submit_operation(dla_processor, dla_common_op_desc, ROI_index)
        (Enter: )
        ('Prepare processor_name, operation index, ROI, dep_count op_desc->dependency_count')
        dla_prepare_operation
        dla_program_operation(processor, processor->group[group_id])
        if dependency_count == 0, dla_enable_operation
        (Exit: )

    8) dla_dequeue_operation(dla_engine, dla_processor)
        'Dequeue next operation of same type from list of operations'
        (Enter: )
        a loop for processing all ROIs 
            if done processing all ROI for current op, load next op
            else, load the same op for next ROI 
                because we break down convolution by 2d-regions 
        ('Dequeue op from processor_name, index, processor_ROI_index')
        dla_get_op_desc 
        dla_submit_operation 
        dla_put_op_desc(consumer)
        (Exit: )

    9) dla_op_completion(dla_processor, dla_processor group)
        //called by dla_handle_events[#12]
        (Enter: )
        **what is this 'STAT_ENABLE'
        ('Completed processor_name, operation index, ROI')
        'Flush stat descriptor to DRAM'
        #if STAT_ENABLE
            dla_data_write(driver_context, task, stat_data_desc)
        #endif
        'get an extra reference '
            dla_get_refcount
        'switch consumer pointer to next group'
        'update dependency graph for this task'
            dla_update_consumer
        (dla_info("%d HWLs done, totally %d layers\n",
				engine->num_proc_hwl,
				engine->network->num_operations); )
        'free operation descriptor from cache' 
            dla_reset_group
        'if all HWLs are done, it means network is complete'
            dla_put_op_desc
        'next group must be ready for programming'
            some kind of a control branch here, branching off into:
        dequeue_op:
            'dequeue operation from this processor-the hw engine'
            dla_dequeue_operation(engine, processor)[see number 8] 
        exit:
            dla_put_op_desc
            (Exit: )

    10) dla_read_network_config(dla_engine)
        (Enter: )
        'Read address list from DRAM to DMEM'
            dla_read_address_list(engine)
        'Read network descriptor address from address list, always at index 0'
            dla_get_dma_address(driver_context, task_data, network_addr)
        'Read network descriptor, it has information for a network, such as all address indexes'
            dla_data_read(driver_context, task, network_addr, DESTINATION_PROCESSOR)
                [this was from KMD portability API doc: http://nvdla.org/sw/runtime_environment.html#c.dla_register_driver
                    'fn to read data from UMD-passed shared buffers into local buffer']
        dla_debug_network_desc
        'Read operation descriptor list address from address list'
        'Read surface descriptor list address from address list'
        'Read dependency graph address from address list'
        'Read LUT data list address from address list'
            (these are all same fns with different input values)
            dla_get_dma_address [also on KMD portabilioty API doc]
        'Read address for ROI info'
            dla_get_dma_address
            dla_data_read
        #if STAT_ENABLE
            dla_get_dma_address : Thus is gets something
        #endif
        (Exit: )

    11) dla_initiate_processors(dla_engine)
        (Enter: )
        'Validate operation heads before initiating processors'
        for each 'operation_head' 
            dla_get_op_desc
            dla_submit_operation
            dla_put_op_desc
            dla_dequeue_operation
        (Exit: )
    
    12) dla_handle_events
        //called by dla_process_events[15]
        (Enter: )
        dunno why, put specific handling of CDMA
            dla_update_consumers(group, op_desc, CDMA_WT_DONE)
            dla_update_consumers(group, op_desc, CDMA_DT_DONE)
        'Handle complete after all other events'
            dla_op_completion(processor, group)
        (Exit: )

**other methods of schedular.c
    13) dla_update_dependency(consumer, op_desc, event, roi_index)
        dla_get_engine()
        'Update dependency operation index, ROI, DEP_COUNT'
        (if op_desc_dependency_count==0)
            'enable processor_name in function_name, as dependency are resolved'
        
    14) dla_update_consumer(group, op, event)
        dla_get_engine()
        // if all branches go accordingly. 
        for i in DPA_OP_NUM:
            dla_update_dependency(op->consumers[i], group->consumers[i], event, group->roi_index)
        ret = dla_update_dependency(op->fused_parent, group->fused_parent, event, group->roi_index)
        if (ret) (Failed to update dependency for fused parent)

    *the below 3 are the basic workflow methods.     
    15) dla_process_events(engine_context, task_complete)
        loop:    
            dla_handle_events(processor)

    dla_execute_task(engine_context, task_data, config_data)
        /**
        * Execute task selected by task scheduler
        *
        * 1. Read network configuration for the task
        * 2. Initiate processors with head of list for same op
        * 3. Start processing events received
        */
        dla_read_network_config [see number 10]
        dla_debug_address_info(task)
        dla_initiate_processor(engine)

    dla_clear_task(engine_context)
        just resetting every reference.. 
    

//**Must target the specific location where nvdla is to be programmed

With diagram from weekly report, 

*the start of Trace is 
dla_execute_task (engine_context, task_data, config_data)  
    /**
        * Execute task selected by task scheduler * 
        *
        * 1. Read network configuration for the task
        * 2. Initiate processors with head of list for same op
        * 3. Start processing events received
    */
    dla_read_network_config[#10](Enter: dla_read_network_config) // DMA to network descriptor
    dla_debug_address_info(task)
    dla_initiate_processors[#11](engine) 
        (Enter: dla_initiate_processors)
        'Validate operation heads before initiating processors'
        for each 'operation_head' 
            dla_get_op_desc
            dla_submit_operation[#7](dla_processor, dla_common_op_desc, ROI_index)
                (Enter: dla_submit_operation)
                'Prepare processor_name, operation index, ROI, dep_count op_desc->dependency_count'
                dla_prepare_operation[#4](dla_processor, dla_common_op_desc, roi_index, group_number) 
                    (Enter: dla_prepare_operation)
                    utils_get_free_group
                    'processor: processor_name, group_id, rdma_group_id, available'
                    //'update operation descriptor'
                    dla_read_config[#3]
                        (Enter: dla_read_config)
                        dla_get_engine //go deeper into 
                        dla_data_read
                        (Exit: dla_read_config)
                    group->pending = 1
                    (Exit: dla_prepare_operation)
                dla_program_operation[#5](processor, processor->group[group_id])
                    (Enter: dla_program_operation) 
                    program trace ('program, operation index, ROI, group_id')
                    group->programming = 1
                    program(group) ********
                    ('prefetch consumers')
                        dla_get_op_desc
                        **Here, fused_parent is also involved "Invalid fused op type warning"
                    dla_op_programmed[#2](dla_processor, dla_proessor_group, rdma_id)
                        (Enter: dla_op_programmed)
                        group->pending = 0
                        dla_update_consumer[other methods](group, op, event)
                            dla_get_engine()
                            // if all branches go accordingly. 
                            for i in DPA_OP_NUM:
                                dla_update_dependency(op->consumers[i], group->consumers[i], event, group->roi_index)
                                    dla_get_engine()
                                    'Update dependency operation index, ROI, DEP_COUNT'
                                    (if op_desc_dependency_count==0)
                                        'enable processor_name in function_name, as dependency are resolved'
                                        dla_enable_operation(processor, op_desc)
                                              dla_enable_operation[#6](dla_processor, dla_common__op_desc)      
                                ret = dla_update_dependency(op->fused_parent, group->fused_parent, event, group->roi_index)
                                if (ret) ('Failed to update dependency for fused parent') 
                        (Exit: dla_op_programmed)
                    (Exit: dla_program_operation)
                (if dependency_count == 0) dla_enable_operation[#6](dla_processor, dla_common__op_desc)
                    (Enter: dla_enable_operation)
                    ('find out if operation if already programmed' , goto enable_op)
                        thus an operation must be 'programmed' before 'enabled'
                     enable_op flow:
                    (if event is triggered as a part of same groups programming, skip enable, as it will be programmed after the current one is complete )
                        ('Enable processor_name, operation index, ROI')
                        enable(group)
                        dla_op_enabled(group)
                    (Exit: dla_enable_operation)
                (Exit: dla_submit_operation)
            dla_put_op_desc
            dla_dequeue_operation
                (Enter: dla_dequeue_operation)
                (if operation index is final for that hardware operation)
                    'exit processor_name as there is no further operation' ~ the end for that hardware layer engine.
                    goto exit;
                'Dequeue op from processor_name, index, ROI'
                 dla_get_op_desc()
                 dla_submit_operation
                 dla_put_op_desc()
                exit:
                    ("Exit: dla_dequeue_operation")  
        (Exit: dla_initiate_processors)

//aside from the call stack above, there is another thread that handles interrupt from hardware
    // called the dla_isr_handler. 


---- 
Then there is handling event Trace:

from nvdla_core_callbacks.c (callbacks are references to executable code that is paased as an argument to another piece of code)
~~so these must be called from UMD? 
    nvdla_engine_isr(irq, data)
        spin_lock_irqsave
        dla_isr_handler
        complete
        spin_unblock_irqrestore
    return IRQ_HANDLED

    nvdla_task_submit(nvdla_dev, task)
        dla_execute_task
        while (1)
            wait_for_completion
            spin_lock_irqsave
            dla_process_events
            spin_unblock_irqrestore
        dla_clear_task
    return err 

dla_isr_handler : will be called on a separate API with dla_process_events(), but will share lock with it, thus become concurrent

dla_process_events(engine_context, task_complete)
    for i in DLA_OP_NUM
        dla_handle_events[#12](processor) 
            (Enter: dla_handle_events)
                //dunno why, put specific handling of CDMA
            dla_update_consumers(group, op_desc, CDMA_WT_DONE)
            dla_update_consumers(group, op_desc, CDMA_DT_DONE)
                'Handle complete after all other events'
            dla_op_completion[#9](processor, group)
                (Enter: dla_op_completion)
                '~~ HWLs done, totally ~ layers'
                dla_dequeue_operation[#8]
                    'Dequeue op from processor_name, index, processor_ROI_index'
            (Exit: dla_handle_events)
