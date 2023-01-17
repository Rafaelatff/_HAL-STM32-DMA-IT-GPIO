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

We can have the calling for the callback function before while loop. It is done using the 'HAL_DMA_RegisterCallback'. Those parameters are:

* DMA_HandleTypeDef \*hdma: &hdma_memtomem_dma2_stream0 (the address for the variable for DMA stream 0);
* HAL_DMA_CallbackIDTypeDef CallbackID: HAL_DMA_XFER_CPLT_CB_ID (the enum for the callback ID (see options next));
* void (\* pCallback)(DMA_HandleTypeDef \*_hdma): &MY_DMA_TC_CB (the address for the function we are going to create (at the bottom of main.c file)).

MY_DMA_TC_CB is short for 'my dma transfer complete callback'. The options for HAL_DMA_CallbackIDTypeDef CallbackID (from stm32fxx_hal_dma.h file):

```c
/** 
  * @brief  HAL DMA Error Code structure definition
  */
typedef enum
{
  HAL_DMA_XFER_CPLT_CB_ID         = 0x00U,  /*!< Full transfer     */
  HAL_DMA_XFER_HALFCPLT_CB_ID     = 0x01U,  /*!< Half Transfer     */
  HAL_DMA_XFER_M1CPLT_CB_ID       = 0x02U,  /*!< M1 Full Transfer  */
  HAL_DMA_XFER_M1HALFCPLT_CB_ID   = 0x03U,  /*!< M1 Half Transfer  */
  HAL_DMA_XFER_ERROR_CB_ID        = 0x04U,  /*!< Error             */
  HAL_DMA_XFER_ABORT_CB_ID        = 0x05U,  /*!< Abort             */
  HAL_DMA_XFER_ALL_CB_ID          = 0x06U   /*!< All               */
}HAL_DMA_CallbackIDTypeDef;
```

I added the private function prototype at the top of the code:

![image](https://user-images.githubusercontent.com/58916022/213006031-0a71090b-b3eb-4977-bb27-8abf43c79b7d.png)

And at the bottom:

```c
/* USER CODE BEGIN 4 */
void MY_DMA_TC_CB(DMA_HandleTypeDef *pHandle){
	// do not implement while loops here
}
/* USER CODE END 4 */
```

And in the while loop, the only difference is that we don't hold the processor during the return of complete data transfered (or max delay time). We don't have the 'HAL_DMA_PollForTransfer' function.

Final code (comments reduced):

```c
int main(void)
{
  /* MCU Configuration--------------------------------------------------------*/
  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* Configure the system clock */
  SystemClock_Config();

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_DMA_Init();
  MX_USART2_UART_Init();
  /* USER CODE BEGIN 2 */
  HAL_DMA_RegisterCallback(&hdma_memtomem_dma2_stream0, HAL_DMA_XFER_CPLT_CB_ID, &MY_DMA_TC_CB);
  /* USER CODE END 2 */

  /* Infinite loop */
   while (1)
  {
     /* USER CODE BEGIN 3 */
	  HAL_DMA_Start_IT(&hdma_memtomem_dma2_stream0, (uint32_t) &led_data[0], (uint32_t) &GPIOA->ODR, 1);
	  // after complete this, we will have an interrupt and no need for the HAL_DMA_PollForTransfer to finish

	  /*delay of 400 msec*/
	  current_ticks = HAL_GetTick();
	  while ((current_ticks +400) >= HAL_GetTick());

	  HAL_DMA_Start_IT(&hdma_memtomem_dma2_stream0, (uint32_t) &led_data[1], (uint32_t) &GPIOA->ODR, 1);
	  // after complete this, we will have an interrupt and no need for the HAL_DMA_PollForTransfer to finish

	  /*delay of 400 msec*/
	  current_ticks = HAL_GetTick();
	  while ((current_ticks +400) >= HAL_GetTick());
  }
  /* USER CODE END 3 */
}
```



