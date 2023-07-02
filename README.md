# Stepper-Motor-Control

## 1 目标
- [ ] 实现PWM控制步进电机
- [ ] 实现编码器反馈速度
- [ ] 实现限位开关(home sensor)

## 2 步骤
### 2.1 实现步进电机的PWM控制（参考江科协代码）
- [x] 步进电机的PWM驱动函数能输入PWM的值作为参数
```
#include "stm32f10x.h"                  // Device header

void PWM_Init(void)
{
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	TIM_InternalClockConfig(TIM2);
	
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStruct.TIM_Period = 100 - 1;      //ARR
	TIM_TimeBaseInitStruct.TIM_Prescaler = 7200 - 1;    //PSC
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2,&TIM_TimeBaseInitStruct);
	
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
	TIM_OCInitStructure.TIM_Pulse = 0;      //CCR
	TIM_OC3Init(TIM2,&TIM_OCInitStructure);
	
	TIM_Cmd(TIM2, ENABLE);
}

void PWM_SetCompare3(uint16_t Compare) //设置占空比
{
	TIM_SetCompare3(TIM2, Compare);
}

void PWM_PrescalerConfig(uint16_t Prescaler) //设置频率
{
	TIM_PrescalerConfig(TIM2, Prescaler, TIM_PSCReloadMode_Update);
}
```

### 2.2 实现固定PWM输入，转动指定角度
- [x] MS1，MS2 低电平，1/8 microstep
- [x] 基本步距角0.9°，360/0.9=400 P/R，所以一圈就是3200 plus （400*8=3200）
```
//PWM输出
/***定时器1主模式***/
void TIM1_config(u32 Cycle)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef  TIM_TimeBaseStructure;
    TIM_OCInitTypeDef  TIM_OCInitStructure;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA|RCC_APB2Periph_TIM1 , ENABLE); 

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_11;                   //TIM1_CH4 PA11
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;             //复用推挽输出
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    TIM_TimeBaseStructure.TIM_Period = Cycle-1;                                                   
    TIM_TimeBaseStructure.TIM_Prescaler =71;                    //设置用来作为TIMx时钟频率除数的预分频值                                                     
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;     //设置时钟分割：TDTS= Tck_tim            
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up; //TIM向上计数模式
    TIM_TimeBaseStructure.TIM_RepetitionCounter = 0;            //重复计数，一定要=0！！！
    TIM_TimeBaseInit(TIM1, &TIM_TimeBaseStructure);                                       

    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;          //选择定时器模式：TIM脉冲宽度调制模式1       
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable; //比较输出使能
    TIM_OCInitStructure.TIM_Pulse = Cycle/2-1;                    //设置待装入捕获寄存器的脉冲值                                   
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_Low;      //输出极性       

    TIM_OC4Init(TIM1, &TIM_OCInitStructure);                                                         

    TIM_SelectMasterSlaveMode(TIM1, TIM_MasterSlaveMode_Enable);
    TIM_SelectOutputTrigger(TIM1, TIM_TRGOSource_Update); 

    TIM_OC4PreloadConfig(TIM1, TIM_OCPreload_Enable);                              
    TIM_ARRPreloadConfig(TIM1, ENABLE);                                                          
}
/***定时器2从模式***/
void TIM2_config(u32 PulseNum)
{
    TIM_TimeBaseInitTypeDef  TIM_TimeBaseStructure;
    NVIC_InitTypeDef NVIC_InitStructure; 
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    TIM_TimeBaseStructure.TIM_Period = PulseNum-1;   
    TIM_TimeBaseStructure.TIM_Prescaler =0;    
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;     
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;  
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);  

    TIM_SelectInputTrigger(TIM2, TIM_TS_ITR0);
    //TIM_InternalClockConfig(TIM2);
    TIM2->SMCR|=0x07;                                  //设置从模式寄存器 
    //TIM_ITRxExternalClockConfig(TIM2, TIM_TS_ITR0);

    //TIM_ARRPreloadConfig(TIM2, ENABLE);         
    TIM_ITConfig(TIM2,TIM_IT_Update,DISABLE);

   // NVIC_PriorityGroupConfig(NVIC_PriorityGroup_3);
    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;        
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;     
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE; 
    NVIC_Init(&NVIC_InitStructure);
}
void Pulse_output(u32 Cycle,u32 PulseNum)
{
    TIM2_config(PulseNum);
    TIM_Cmd(TIM2, ENABLE);
    TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
    TIM_ITConfig(TIM2,TIM_IT_Update,ENABLE);
    TIM1_config(Cycle);
    
    TIM_Cmd(TIM1, ENABLE);
    TIM_CtrlPWMOutputs(TIM1, ENABLE);   //高级定时器一定要加上，主输出使能
}

void TIM2_IRQHandler(void) 
{ 
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)     // TIM_IT_CC1
    { 
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update); // 清除中断标志位 
        TIM_CtrlPWMOutputs(TIM1, DISABLE);  //主输出使能
        TIM_Cmd(TIM1, DISABLE); // 关闭定时器 
        TIM_Cmd(TIM2, DISABLE); // 关闭定时器 
        TIM_ITConfig(TIM2, TIM_IT_Update, DISABLE);   
    } 
}
```
### 2.3 串口通讯（参考江科协代码）
- [x] 串口输出热敏电阻的温度值，连接上位机图形化显示 
```Serial_Printf("Rps: %.2f\n", Rps); //串口输出转速，换行打印```
- [ ] 待完成功能：Kp, Ki, Kd参数通过串口输入，而不用每次更改都编译和下载一遍

### 2.4 PID控制温度
- [x] 简单粗暴的控制方式，不能让温度稳定在 xxx 度
```
if (Temp > 41) Speed--;
if (Temp < 39) Speed++;
```
![image](https://github.com/Kevinyym/Heater-PID-Control/assets/101639215/d334016d-43eb-46fe-8963-eefc913e0f4d)

- [x] 热敏电阻PID控制加热片温度
```
float Err=0, LastErr=0, NextErr=0, Add=0, Kp=10, Ki=0.6, Kd=0.05, POut=0, IOut=0, DOut;
int8_t TotalOut=0;

/**
  * @brief  PID控制温度
  * @param  Rps: 将当前的目标值传入函数
  * @param  Target: 将当前的测量值传入函数
  * @retval 返回执行量
  */
float PID(float Rps, float Target)
{
	Err = Target - Rps; //计算实际值与目标值的偏差值
	POut = Kp * Err; //计算PID的比例值P的输出值
	IOut += Ki * Err; //计算PID的积分值I的输出值
	DOut = Err - LastErr; //计算PID的微分值D的输出值
	TotalOut = (int8_t)(POut + IOut + DOut); //PID值的和
	LastErr = Err;
	return TotalOut;
}
```
![image](https://github.com/Kevinyym/Heater-PID-Control/assets/101639215/5ffb76e7-1444-4fa3-ac33-8ee9707dbb5d)

## 3 定时器表资源表：STM32F103C8T6定时器资源：TIM1,TIM2,TIM3,TIM4
```	
	*功能*	  *定时器*	 *类型*
	闲置	   TIM1	   高级定时器
	PWM	    TIM2    通用定时器	
	(Encoder)   TIM3    通用定时器	
	(Timer)	    TIM4    通用定时器	
```
## 4 接线框图
- [ ] 电机驱动TMC2209（RX，TX，CLK可以不接）注意：EN脚一定要接低电平，悬空电机不转，切记
- [ ] 电容100uf，接在电机电源上
- [ ] 开关电源12V（5V带不起来）
- [ ] 步进电机PKP243MD15A-R2FL, 编码器分辨率400P/R，基本步距角0.9°

![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/f446d52d-bf39-4ad2-bb1f-805620771ffd)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/8bf0b0a4-6273-4ea5-9191-296d6575488c)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/370dd96f-9677-4c7b-b5ef-e6f2c3c48022)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/f48e4e6a-3e98-4126-8cd8-86b325d3d6dd)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/df2eb117-ac03-457a-b58a-351279d90b28)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/565eb096-2b27-4e59-a8e0-85043c7a0f56)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/904a075d-63a5-4789-91b6-4eb49f1b498e)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/0081eeea-2529-4660-a29b-c055695b40a0)


## 5 参考资料
- [A4988驱动NEMA步进电机(42步进电机)](http://www.taichi-maker.com/homepage/reference-index/motor-reference-index/arduino-a4988-nema-stepper-motor/)
- [PKP243MD15A-R2FL,高分辨率型 带编码器（双极性 4根导线）](https://www.orientalmotor.com.cn/products/st/list/detail/?product_name=PKP243MD15A-R2FL&brand_tbl_code=ST)
- [VORON/klipper 如何使用TMC2209以及使用无传感器归零功能 (一) 硬件介绍与连接](https://blog.csdn.net/weixin_43234123/article/details/123128094)
- [嵌入式STM32学习笔记（5）——定时器主从模式，精确输出PWM脉冲数量](https://blog.csdn.net/abcvincent/article/details/95250994?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-95250994-blog-96273988.235^v38^pc_relevant_sort&spm=1001.2101.3001.4242.2&utm_relevant_index=4)
- [各种方案，精确输出可控脉冲个数，尽量可控周期或占空比（脉冲 dma](http://www.openedv.com/forum.php?mod=viewthread&tid=297375)
- [STM32主从模式 精确脉冲数PWM （已实现）](https://blog.csdn.net/leonsust/article/details/96273988)
