<img width="803" height="527" alt="image" src="https://github.com/user-attachments/assets/daa4cefe-5099-4d5f-9202-26121dd8cb8d" /><img width="692" height="478" alt="image" src="https://github.com/user-attachments/assets/a1c801fd-ad80-4483-9057-6c7e007da38a" />首先是 __HAL_FLASH_PREFETCH_BUFFER_ENABLE();设置FLASH->ACR寄存器，位4为1，启用预取缓冲区PRFTBE
FLASH->ACR |= FLASH_ACR_PRFTBE（0x1<<4）
其次设置NVIC中断
void HAL_NVIC_SetPriorityGrouping(uint32_t PriorityGroup)
{
  /* Check the parameters */
  assert_param(IS_NVIC_PRIORITY_GROUP(PriorityGroup)); //断言函数，调试时候使用
  
  /* Set the PRIGROUP[10:8] bits according to the PriorityGroup parameter value */
  NVIC_SetPriorityGrouping(PriorityGroup);
}
使用断言函数应注意一下两点：1.取消注释 2.打印错误
#define USE_FULL_ASSERT    1

void assert_failed(uint8_t* file, uint32_t line)
{
 printf("Wrong parameters value: file %s on line %d\r\n", file, line);
 while(1);
}

__STATIC_INLINE void __NVIC_SetPriorityGrouping(uint32_t PriorityGroup)
{
  uint32_t reg_value;
  uint32_t PriorityGroupTmp = (PriorityGroup & (uint32_t)0x07UL);             /* only values 0..7 are used  取出中断优先级的后3位  00000003 & 0x00000007 =  0000 0003      */

  reg_value  =  SCB->AIRCR;                                                   /* read old register configuration  此时SCB->AIRCR=0xFA05 0000   */
  reg_value &= ~((uint32_t)(SCB_AIRCR_VECTKEY_Msk | SCB_AIRCR_PRIGROUP_Msk)); /* clear bits to change    0xFA050000 & ~(0xFFFF0000 | 0x700) = 0xFA050000 & 0x0000F8FF = 0x0000 0000           */
  reg_value  =  (reg_value                                   |
                ((uint32_t)0x5FAUL << SCB_AIRCR_VECTKEY_Pos) |
                (PriorityGroupTmp << SCB_AIRCR_PRIGROUP_Pos) );               /* Insert write key and priority group 0x05FA<<16 = 0x05FA00000 0x00000003<<8=0x00000300  0x05FA00000 | 0x00000300 = 0x05FA 0300*/
  SCB->AIRCR =  reg_value;
}
SCB系统控制模块设置AIRCR寄存器位31-16为0x05FA，位10-8为011
然后HAL_InitTick(TICK_INT_PRIORITY);

__weak HAL_StatusTypeDef HAL_InitTick(uint32_t TickPriority)
{
  /* Configure the SysTick to have interrupt in 1ms time basis*/
  if (HAL_SYSTICK_Config(SystemCoreClock / (1000U / uwTickFreq)) > 0U)    //频率为16MHz,1s数16M个数，1ms数(16M/1000)个数
  {
    return HAL_ERROR;
  }

  /* Configure the SysTick IRQ priority */
  if (TickPriority < (1UL << __NVIC_PRIO_BITS))
  {
    HAL_NVIC_SetPriority(SysTick_IRQn, TickPriority, 0U);
    uwTickPrio = TickPriority;
  }
  else
  {
    return HAL_ERROR;
  }

  /* Return function status */
  return HAL_OK;
}

uint32_t HAL_SYSTICK_Config(uint32_t TicksNumb)
{
   return SysTick_Config(TicksNumb);
}

__STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)
{
  if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)  //判断16000-1 是否大于 0xFFFFFF，若大于，则出错
  {
    return (1UL);                                                   /* Reload value impossible */
  }

  SysTick->LOAD  = (uint32_t)(ticks - 1UL);                         /* set reload register 将1ms计的数16000放入重装载寄存器 */   
  NVIC_SetPriority (SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL); /* set Priority for Systick Interrupt  SysTick_IRQn = -1（0xFFFFFFFF）(1<<4)-1=15 */
  SysTick->VAL   = 0UL;                                             /* Load the SysTick Counter Value */
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                   SysTick_CTRL_TICKINT_Msk   |
                   SysTick_CTRL_ENABLE_Msk;                         /* Enable SysTick IRQ and SysTick Timer */
  return (0UL);                                                     /* Function successful */
}

__STATIC_INLINE void __NVIC_SetPriority(IRQn_Type IRQn, uint32_t priority)    //__NVIC_SetPriority(-1,15)
{
  if ((int32_t)(IRQn) >= 0)
  {
    NVIC->IP[((uint32_t)IRQn)]               = (uint8_t)((priority << (8U - __NVIC_PRIO_BITS)) & (uint32_t)0xFFUL);
  }
  else
  {
    SCB->SHP[(((uint32_t)IRQn) & 0xFUL)-4UL] = (uint8_t)((priority << (8U - __NVIC_PRIO_BITS)) & (uint32_t)0xFFUL);  //(1111<<4) & 0xFF = 0xF0
  }
}



