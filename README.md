# Stepper-Motor-Control

## 1 目标
- [x] 实现PWM控制步进电机
- [x] 实现输出精确脉冲数（定时器主从模式）
- [x] 实现步进电机梯形加减速控制
- [x] 实现编码器反馈速度
- [x] 实现串口上位机显示速度
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
- [x] 将2.1中的TIM2单独输出PWM，改成TIM1和TIM2进行定时器主从模式来精确输出脉冲数
- [x] MS1，MS2 低电平，1/8 microstep
- [x] 基本步距角0.9°，360/0.9=400 P/R，所以一圈就是3200 plus （400*8=3200 plus）
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
### 2.3 实现步进电机梯形加减速控制
- [x] S型曲线加减速, [链接](https://www.cnblogs.com/Tuple-Joe/p/13533324.html)
- [x] Sigmoid函数是一个在生物学中常见的S型函数，也称为S型生长曲线。Sigmoid函数也叫Logistic函数，取值范围为(0,1)，它可以将一个实数映射到(0,1)的区间，可以用来做二分类。该S型函数有以下优缺点：优点是平滑，而缺点则是计算量大。
- [x] ![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/7bded38b-2635-4ac6-97f8-78790b27f68d)
- [x] ![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/616e7baf-1cd0-493d-9824-fdad5f3b4896)
- [x] 在电机加减速控制上，电机频率越大，电机速度越快。因而，可以通过公式法求出每个加减速点的频率值，进而通过电机频率求出具体的脉冲周期，最后在间隔相同的时间内改变脉冲相关参数（分频、周期、占空比）即可达到加减速的效果。一般情况下，如步进电机、伺服电机等，分频(71)与占空比(50%)通常固定数值即可，这样在加减速过程仅需改变输出周期值即可。
```
#define Len 20 //宏定义S曲线的变速脉冲点
float Fre[Len];
u32 Period[Len]; 
void Calculate_S_Curve(float StartFre, float EndFre, float Flexible)
{
	u32 TIM_CLK = 72000000;
	u8 TIM_FRESCALER = 71;
    float melo;

    for(int i = 0; i < Len; i ++)
    {
        melo = Flexible * (i-Len/2.0) / (Len/2.0);
        Fre[i] = StartFre + (EndFre - StartFre) / (1 + expf(-melo));
        Period[i] = (u32)(TIM_CLK / TIM_FRESCALER / Fre[i]);       // TIM_CLK 定时器时钟   TIM_FRESCALER 定时器分频
	}
}
```
- [x] 同时，不同频率脉冲输出时也需要注意脉冲的连续性（即我们需要在当前脉冲完全输出之后才能改变电机频率），否则电机加减速过程就会出现丢步现象，在脉冲数严格要求的情况下造成累积误差。main函数中代码如下：先计算出脉冲频率变更点，然后for循环调用频率数组，while循环检查中断标志，确保在当前脉冲完全输出之后才能改变电机频率
```
Calculate_S_Curve(1000, 4000, 3); //公式计算S曲线
for (int i=0; i<Len; i++)
{
	Pulse_output(Period[i],80); //变速脉冲点的ARR每固定脉冲数后改变
	while (IRQ_Flag == 0)
		{
			OLED_ShowNum(4,5,i,2);
			OLED_ShowNum(4,9,Fre[i],5);
		}
	IRQ_Flag = 0;
}
Pulse_output(Period[(Len-1)],3200*3);
```
- [x] Calculate_S_Curve函数，用Sublime Text 输出具体脉冲频率值后，导入execel，如下
- [x] ![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/41f47180-1b36-46fd-8fb6-8503ee1ad48b)
- [x] 用Freq单独做表，如下变化
- [x] ![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/9552713c-af0b-483d-ad2c-01b2ac0d8034)
- [x] 用Time-Freq做表，如下， Time是Freq的倒数，每个频率输出80个脉冲，所以Time*80
- [x] ![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/d6de82a0-25b8-4893-ba42-06480298c9ff)
- [ ] 遗留问题：加速过程电机发生低频振动。可能解决方案：从现有8细分改成更大的细分，比如64

### 2.4 实现编码器反馈速度
- [x] 编码器是ABZ型，此应用中只接A+/B+信号，相位差=P/4,所以4个信号代表1个脉冲

```
#include "stm32f10x.h"                  // Device header

void Encoder_Init(void)
{
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStruct.TIM_Period = 65536 - 1;      //ARR
	TIM_TimeBaseInitStruct.TIM_Prescaler = 1 - 1;    //PSC
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM3,&TIM_TimeBaseInitStruct);
	
	TIM_ICInitTypeDef TIM_ICInitStructure;
	TIM_ICStructInit(&TIM_ICInitStructure);
	TIM_ICInitStructure.TIM_Channel = TIM_Channel_1;
	TIM_ICInitStructure.TIM_ICFilter = 0xF;
	TIM_ICInit(TIM3, &TIM_ICInitStructure);
	TIM_ICInitStructure.TIM_Channel = TIM_Channel_2;
	TIM_ICInitStructure.TIM_ICFilter = 0xF;
	TIM_ICInit(TIM3, &TIM_ICInitStructure);	
	
	TIM_EncoderInterfaceConfig(TIM3, TIM_EncoderMode_TI12, TIM_ICPolarity_Rising, TIM_ICPolarity_Rising);

	TIM_Cmd(TIM3, ENABLE);
}

int16_t Encoder_Get(void)
{
	int16_t Temp;
	Temp = TIM_GetCounter(TIM3);
 	TIM_SetCounter(TIM3, 0);
	return Temp;
}
```
### 2.5 串口通讯（参考江科协代码）
- [x] 串口输出转速，连接上位机图形化显示
- [ ] 加速还需优化，考虑使用24V开关电源（电机额定24V，目前使用12v开关电源）
- [ ] 
```Serial_Printf("Speed:%f\n", Speed); //串口输出转速，换行打印```
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/f3de2852-ae23-4c99-9cbf-642e6e01744a)

## 3 定时器表资源表：STM32F103C8T6定时器资源：TIM1,TIM2,TIM3,TIM4
```	
	*功能*	  *定时器*	 *类型* 		*备注*
	PWM	   TIM1	   	高级定时器	步进电机控制
	脉冲计数	   TIM2    	通用定时器	输出精确脉冲数
	Encoder    TIM3    	通用定时器	编码器脉冲计数
	Timer	    TIM4    	通用定时器	监控按键
```
## 4 接线框图
- [ ] 电机驱动TMC2209（RX，TX，CLK可以不接）注意：EN脚一定要接低电平，悬空电机不转，切记
- [ ] 电容100uf，接在电机电源上
- [ ] 开关电源12V（5V带不起来）
- [ ] 步进电机PKP243MD15A-R2FL, 编码器分辨率400P/R，基本步距角0.9°. 接头Pin#：1248，分别对应GND/A+/B+/VCC（DC5V），从ST-Link直接取5V和GND），编码器输出AB相接在PA6和PA7两个GPIO口

![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/3ed4a86a-a551-409f-afa9-8007267623a0)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/8bf0b0a4-6273-4ea5-9191-296d6575488c)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/370dd96f-9677-4c7b-b5ef-e6f2c3c48022)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/f48e4e6a-3e98-4126-8cd8-86b325d3d6dd)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/e031662d-e14f-4db1-8d05-492dda5c1812)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/565eb096-2b27-4e59-a8e0-85043c7a0f56)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/904a075d-63a5-4789-91b6-4eb49f1b498e)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/0081eeea-2529-4660-a29b-c055695b40a0)
![image](https://github.com/Kevinyym/Stepper-Motor-Control/assets/101639215/b9390a54-9276-4e4f-9f5f-94b93f9851bc)


## 5 参考资料
- [A4988驱动NEMA步进电机(42步进电机)](http://www.taichi-maker.com/homepage/reference-index/motor-reference-index/arduino-a4988-nema-stepper-motor/)
- [PKP243MD15A-R2FL,高分辨率型 带编码器（双极性 4根导线）](https://www.orientalmotor.com.cn/products/st/list/detail/?product_name=PKP243MD15A-R2FL&brand_tbl_code=ST)
- [VORON/klipper 如何使用TMC2209以及使用无传感器归零功能 (一) 硬件介绍与连接](https://blog.csdn.net/weixin_43234123/article/details/123128094)
- [嵌入式STM32学习笔记（5）——定时器主从模式，精确输出PWM脉冲数量](https://blog.csdn.net/abcvincent/article/details/95250994?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-95250994-blog-96273988.235^v38^pc_relevant_sort&spm=1001.2101.3001.4242.2&utm_relevant_index=4)
- [各种方案，精确输出可控脉冲个数，尽量可控周期或占空比（脉冲 dma](http://www.openedv.com/forum.php?mod=viewthread&tid=297375)
- [STM32主从模式 精确脉冲数PWM （已实现）](https://blog.csdn.net/leonsust/article/details/96273988)
- [正交编码 正交编码器 增量式编码器](https://blog.csdn.net/huangbinvip/article/details/123962511)
