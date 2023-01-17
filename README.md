# _HAL-STM32-DMA-IT-GPIO
This repository was create to follow the content of the course 'ARM Cortex M Microcontroller DMA Programming Demystified'.  

I am using:  
* NUCLEO-F401RE Board 
* STM32Cube FW_F4 V1.27.1 
* STM32CubeIDE version 1.10.1

## STM32CubeMX configuration

This is a continuation from the [_HAL-STM32-DMA-Poll-GPIO](https://github.com/Rafaelatff/_HAL-STM32-DMA-Poll-GPIO), but instead of using polling methode, we will be using interrupt.

The only difference from the previous project, will be in those settings (we enable interrupt for DMA):

![image](https://user-images.githubusercontent.com/58916022/212998105-d2c81785-515c-40da-a7f3-316475f56dae.png)

![image](https://user-images.githubusercontent.com/58916022/212998170-77e4d576-b515-423b-9cc3-3346980e064f.png)

# Code

With this configuration, we will have in the 'stm32f4xx_it.c' file the IRQ's: 'DMA2_Stream0_IRQHandler' and 'SysTick_Handler'.

The 'DMA2_Stream0_IRQHandler' IRQ handler, will then, call the *HAL_DMA_IRQHandler(DMA_HandleTypeDef \*hdma)* function. This functions pass through all the registers of the DMA and then, according with the flags/erros set, it will return the callback functions.

We declared a global variable to the data to be send to port A:

```c
uint8_t led_data[2]= { 0x00, 0xff};
``` 

Then we generate a DMA start (with IT):

```c
	  HAL_DMA_Start_IT(&hdma_memtomem_dma2_stream0, (uint32_t) &led_data[0], (uint32_t) &GPIOA->ODR, 1);
```

After complete this start, we will have an interrupt that goes to 'DMA2_Stream0_IRQHandler'. 

**TIP: inside the 'HAL_DMA_IRQHandle', add breakpoints after the *if*'s to be aware of what callback function will be called**.

In our application, the 'Half Transfer Complete Interrupt managmente' was called before the 'Transfer Complete Interrupt management'. We can check in the memory addredd the data copied to the GPIOA->ODR address. If the callback isn't initialized, it will be NULL and won't call the callback function.

Bassicaly (from _HAL driver):

```c
      if(((hdma->Instance->CR) & (uint32_t)(DMA_SxCR_DBM)) != RESET)
      {
        /* Current memory buffer used is Memory 0 */
        if((hdma->Instance->CR & DMA_SxCR_CT) == RESET)
        {
          if(hdma->XferM1CpltCallback != NULL)
          {
            /* Transfer complete Callback for memory1 */
            hdma->XferM1CpltCallback(hdma);
          }
        }
        /* Current memory buffer used is Memory 1 */
        else
        {
          if(hdma->XferCpltCallback != NULL)
          {
            /* Transfer complete Callback for memory0 */
            hdma->XferCpltCallback(hdma);
          }
        }
      }
```

