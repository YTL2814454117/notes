<!--
 * @Author: 睦疏 
 * @Date: 2023-04-03 19:48:33
 * @LastEditors: 睦疏
 * @LastEditTime: 2023-04-03 22:19:37
 * @FilePath: \notes\硬件组学习\oled驱动学习.md
 * @Description: 
 * 
 * Copyright (c) 2023 by ${git_name_email}, All Rights Reserved. 
-->

# I2C软件实现

``` c
// 起始信号
void I2C_Start(void)
{
	OLED_SDA_Set();
	OLED_SCL_Set();
	IIC_delay();
	OLED_SDA_Clr();
	IIC_delay();
	OLED_SCL_Clr();
	IIC_delay();
}
// 结束信号
void I2C_Stop(void)
{
	OLED_SDA_Clr();
	OLED_SCL_Set();
	IIC_delay();
	OLED_SDA_Set();
}

// 等待信号响应
void I2C_WaitAck(void) // 测数据信号的电平
{
	OLED_SDA_Set();
	IIC_delay();
	OLED_SCL_Set();
	IIC_delay();
	OLED_SCL_Clr();
	IIC_delay();
}
```

``` c
// 写入一个字节
void Send_Byte(u8 dat)
{
	u8 i;
	for (i = 0; i < 8; i++)
	{
		if (dat & 0x80) // 将dat的8位从最高位依次写入
		{
			OLED_SDA_Set();
		}
		else
		{
			OLED_SDA_Clr();
		}
		IIC_delay();
		OLED_SCL_Set();
		IIC_delay();
		OLED_SCL_Clr(); // 将时钟信号设置为低电平
		dat <<= 1;
	}
}

// 发送一个字节
// mode:数据/命令标志 0,表示命令;1,表示数据;
void OLED_WR_Byte(u8 dat, u8 mode)
{
	I2C_Start();
	Send_Byte(0x78);
	I2C_WaitAck();
	if (mode)
	{
		Send_Byte(0x40);
	}
	else
	{
		Send_Byte(0x00);
	}//判断是命令还是数据 让oled屏幕知道
	I2C_WaitAck();
	Send_Byte(dat);//写入一个字节
	I2C_WaitAck();
	I2C_Stop();
}
```
# oled数据处理
``` c
// 开启OLED显示
void OLED_DisPlay_On(void)
{
	OLED_WR_Byte(0x8D, OLED_CMD); // 电荷泵使能
	OLED_WR_Byte(0x14, OLED_CMD); // 开启电荷泵
	OLED_WR_Byte(0xAF, OLED_CMD); // 点亮屏幕
}

// 关闭OLED显示
void OLED_DisPlay_Off(void)
{
	OLED_WR_Byte(0x8D, OLED_CMD); // 电荷泵使能
	OLED_WR_Byte(0x10, OLED_CMD); // 关闭电荷泵
	OLED_WR_Byte(0xAE, OLED_CMD); // 关闭屏幕
}

//oled屏幕中的电荷泵用途

--------

**ChatGPT**:

在 OLED（Organic Light Emitting Diode）屏幕中，电荷泵（Charge Pump）是一种用于生成高电压的电路。
它的主要作用是将低电压直流电源（如3.3V或5V）转换为高电压直流电源（如15V或20V），以便驱动 OLED 屏幕
中的发光二极管。OLED 屏幕中的每个像素都包含了一个发光二极管和一个薄膜晶体管（TFT），这些元件需要被
精确地驱动才能产生高质量的图像。而要精确驱动这些元件，就需要高电压直流电源。电荷泵能够通过将电荷转移
来实现高电压的生成。当电荷从一个电容器转移到另一个电容器时，电压也会跟随变化。通过交替地将电荷从一个
电容器转移到另一个电容器，电荷泵可以实现高电压的生成。因此，电荷泵在 OLED 屏幕中非常重要，它能够为屏
幕提供所需的高电压，以产生高质量的图像。
```

``` c
// 更新显存到OLED
void OLED_Refresh(void)
{
	u8 i, n;
	for (i = 0; i < 4; i++)
	{
		OLED_WR_Byte(0xb0 + i, OLED_CMD); // 设置行起始地址
		OLED_WR_Byte(0x00, OLED_CMD);	  // 设置低列起始地址0
		OLED_WR_Byte(0x10, OLED_CMD);	  // 设置高列起始地址16 一共32列
		I2C_Start();
		Send_Byte(0x78);
		I2C_WaitAck();
		Send_Byte(0x40);
		I2C_WaitAck();
		for (n = 0; n < 128; n++)
		{
			Send_Byte(OLED_GRAM[n][i]);
			I2C_WaitAck();
		}//写入显存中的四列像素数据到oled屏幕
		I2C_Stop();
	}
}

// 清屏函数
void OLED_Clear(void)
{
	u8 i, n;
	for (i = 0; i < 4; i++)
	{
		for (n = 0; n < 128; n++)
		{
			OLED_GRAM[n][i] = 0; // 清除所有数据
		}
	}
	OLED_Refresh(); // 更新显示
}

```

# oled画画函数
``` c
// 画点
// x:0~127
// y:0~63
// t:1 填充 0,清空
void OLED_DrawPoint(u8 x, u8 y, u8 t)
{
	u8 i, m, n;
	i = y / 8;//y的最大值为31 除8之后分配显存更合适
	m = y % 8;//y取模结果作为n左移的位数
	n = 1 << m;//1左移m位
	if (t)
	{
		OLED_GRAM[x][i] |= n; //按位或后赋值
	}
	else
	{
		OLED_GRAM[x][i] = ~OLED_GRAM[x][i];//~是取反符号
		OLED_GRAM[x][i] |= n;
		OLED_GRAM[x][i] = ~OLED_GRAM[x][i];
	}
}

// 画线
// x1,y1:起点坐标
// x2,y2:结束坐标
void OLED_DrawLine(u8 x1, u8 y1, u8 x2, u8 y2, u8 mode)
{
	u16 t;
	int xerr = 0, yerr = 0, delta_x, delta_y, distance;
	int incx, incy, uRow, uCol;
	delta_x = x2 - x1; // 计算坐标增量
	delta_y = y2 - y1;
	uRow = x1; // 画线起点坐标
	uCol = y1;
	if (delta_x > 0)
		incx = 1; // 设置单步方向
	else if (delta_x == 0)
		incx = 0; // 垂直线
	else
	{
		incx = -1;
		delta_x = -delta_x;
	}
	if (delta_y > 0)
		incy = 1;
	else if (delta_y == 0)
		incy = 0; // 水平线
	else
	{
		incy = -1;
		delta_y = -delta_x;
	}
	if (delta_x > delta_y)
		distance = delta_x; // 选取基本增量坐标轴
	else
		distance = delta_y;
	for (t = 0; t < distance + 1; t++)
	{
		OLED_DrawPoint(uRow, uCol, mode); // 画点
		xerr += delta_x;
		yerr += delta_y;
		if (xerr > distance)
		{
			xerr -= distance;
			uRow += incx;
		}
		if (yerr > distance)
		{
			yerr -= distance;
			uCol += incy;
		}
	}
}
// x,y:圆心坐标
// r:圆的半径
void OLED_DrawCircle(u8 x, u8 y, u8 r)
{
	int a, b, num;
	a = 0;
	b = r;
	while (2 * b * b >= r * r)
	{
		OLED_DrawPoint(x + a, y - b, 1);
		OLED_DrawPoint(x - a, y - b, 1);
		OLED_DrawPoint(x - a, y + b, 1);
		OLED_DrawPoint(x + a, y + b, 1);

		OLED_DrawPoint(x + b, y + a, 1);
		OLED_DrawPoint(x + b, y - a, 1);
		OLED_DrawPoint(x - b, y - a, 1);
		OLED_DrawPoint(x - b, y + a, 1);

		a++;
		num = (a * a + b * b) - r * r; // 计算画的点离圆心的距离
		if (num > 0)
		{
			b--;
			a--;
		}
	}
}
```
