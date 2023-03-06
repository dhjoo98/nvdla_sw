the firmware of NVDLA is in charge of the operation of each hardware engine and core scheduling task.
In headed implementation, it is to be located on the nvdla microprocessor, 
In headless implementation, it is included as a part of the kernel. 

cf) shift+ctrl+f searches the entire folder directory in visual studios code 

function descriptions partly from 
http://nvdla.org/sw/runtime_environment.html#c.dla_register_driver

---
Core Engine Interface
main KMD functionalities, some are called from the portability layer, 
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

dla_read_dma_address
    Read DMA address from address list 
    Used by function block programming operations to read address for DMA engines(that each hardware engines have). 
    calls nvdla_gem_dma_addr fn (/port/linux/nvdla_gem.c), which returns a memory handle
    nvdla_gem_dma_addr goes looks up GEM objects

    But what is this 'DMA address' used for? 
        DMA in the NVDLA scope refers to that data transaction from DRAM/SRAM to Buffer of each hardware engine
        Thus the management of shared memory region. 

dla_read_cpu_address 
    Read CPU accessible address 
    Used by engine schedular to read data from memory buffer.

dla_data_read
    

---
Function Block Programming 
so when are the individual engine firmware called? Schedular classifies task, then pass on to somewhere? 

"conv.c" &  other engine firmwares. 



