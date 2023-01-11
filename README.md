# kessy_handles_adapter
## Адаптер для подключения ручек Kessy VW Passat к нештатной сигнализации (или куда-то ещё)

Сенсор ручки имеет выход с открытым коллектором. При подключении к питанию 5В через резистор номиналом 300 Ом и касании разных сенсорных зон на выходе ручки появляются
прямоугольные импульсы с уровнем лог.1 около 4В. Подав эти импульсы на положительный вход ОУ MCP602, а на отрицательный опорное напряжение 4.5В, с выхода ОУ можно снять
следующие осциллограммы:

<p align="center">
<img width="600" src="https://user-images.githubusercontent.com/62425725/211800740-a8c49187-b3dc-4356-9fde-80e9ff5ab4b8.jpg" alt="касание зоны закрытия">
<br>
  <i>касание зоны закрытия</i>
<br>
<img width="600" src="https://user-images.githubusercontent.com/62425725/211800871-fba33fb3-7f2f-4712-a680-137c7604a597.jpg" alt="касание зоны открытия">
<br>
  <i>касание зоны открытия</i>
</p>

Управляющий МК - STM32F103C8T6 (отладочная плата BluePill), импульсы поступают на вход таймера TIM2. Канал 1 таймера настроен на нисходящий фронт сигнала, канал 2 - на
восходящий. Таким образом, измерив время между нисходящим и восходящим сигналом мы получаем время импульса, а между двумя нисходящими - их период. По длительности импульса мы можем определить, какой зоны мы коснулись, и подать сигнал на соответствующий выход МК. Для тестов на выход команды закрытия подключен красный светодиод,
на выход канала открытия жёлтый.

Параметры времени определены в файле main.h:

```
#define Lock_Pulse_Width_ms   41    -   длительность импульса при касании зоны закрытия
#define Unlock_Pulse_Width_ms 8     -   длительность импульса при касании зоны открытия
#define Pause_width_ms        10    -   длительность паузы между импульсами
#define Tolerance_ms          2     -   допуск 
```
Таймер TIM3 считает постоянно и сбрасывает состояние выходов открытия и закрытия в 0 каждые 100 мс, если не обнаружено касание. Это осуществляется в callback-функции:
```
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	if (htim->Instance == TIM3)
	{
		HAL_GPIO_WritePin(Unlock_GPIO_Port, Unlock_Pin, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(Lock_GPIO_Port, Lock_Pin, GPIO_PIN_RESET);
	}
}
```
Обработка событий захвата импульсой таймером TIM2 происходит в другой callback-функции:
```
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM2)
  {
    if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
    {
      period = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_1);
      pulseWidth = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_2);
      
      TIM2->CNT = 0;
			
			if (pulseWidth >= (Lock_Pulse_Width_ms - Tolerance_ms) * 100 && 
				pulseWidth <= (Lock_Pulse_Width_ms + Tolerance_ms) * 100 &&
				(period - pulseWidth) >= (Pause_width_ms - Tolerance_ms) * 100 && 
				(period - pulseWidth) <= (Pause_width_ms + Tolerance_ms) * 100)
		{
			HAL_GPIO_WritePin(Lock_GPIO_Port, Lock_Pin, GPIO_PIN_SET);
			TIM3->CNT = 0;
		}
		
		if (pulseWidth >= (Unlock_Pulse_Width_ms - Tolerance_ms) * 100 && 
				pulseWidth <= (Unlock_Pulse_Width_ms + Tolerance_ms) * 100 &&
				(period - pulseWidth) >= (Pause_width_ms - Tolerance_ms) * 100 && 
				(period - pulseWidth) <= (Pause_width_ms + Tolerance_ms) * 100)
		{
			HAL_GPIO_WritePin(Unlock_GPIO_Port, Unlock_Pin, GPIO_PIN_SET);
			TIM3->CNT = 0;
		}		
    }
  }
}
```
## Видео работы:
<p align="center">
  <a href="https://youtu.be/8dX6zanowjw"><img width="600" src="https://user-images.githubusercontent.com/62425725/211806185-8d0d523a-b263-43f1-8bc6-23b291e46e6c.jpg" alt="youtube">
</p>
