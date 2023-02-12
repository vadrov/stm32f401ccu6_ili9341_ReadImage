Copyright (C) 2023, VadRov, all right reserved.
# stm32f401ccu6_ili9341_ReadImage
 The project demonstrates reading pixel color data from the ili9341 display controller memory into a buffer for further processing or saving a background image before displaying a sprite.
 
 Проект демонстрирует чтение из памяти контроллера дисплея ili9341 данных о цвете пикселей в буфер для последующей обработки или сохранения образа фона перед выводом спрайта.
 
 Схема подключения:
 
 ![stm32 + ili9341 чтение запись схема подключения](https://user-images.githubusercontent.com/111627147/218332686-c31b97b0-825f-4a06-a27b-136516d75d08.jpg)

Картинка работы проекта:

![картинка работы примера](https://user-images.githubusercontent.com/111627147/218332943-08a62d9b-a129-4635-b699-f1c1acf6a2e7.jpg)

Вот так реализована функция чтения памяти контроллера дисплея (опирается на функционал драйвера дисплея). Готовых решений в интернете не нашел, писал по спецификации... А она оказалась откровенным фуфлом. Все коммментарии в коде. Применялся "обратный инжиниринг", опыт и много мата.
```
//Читает данные из окна дисплея с координатами левого верхнего угла (x, y), шириной w, высотой h в буфер data
void LCD_ReadImage(LCD_Handler* lcd, uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint16_t *data)
{
    uint16_t x1 = x + w - 1, y1 = y + h - 1; //Определяем координаты правого нижнего угла окна
    //Проверяем параметры окна, т.к. за пределами размерности дисплея не работаем
    if (x >= lcd->Width || y >= lcd->Height || x1 >= lcd->Width || y1 >= lcd->Height) return;
    //Пересчет координат окна в зависимости от характеристик матрицы дисплея и контроллера.
    //Соответствующий offs в драйвере дисплея определяет эти характеристики, учитывая разницу размерностей,
    //начальное смещение матрицы дисплея относительно поля памяти контроллера дисплея и ориентацию изображения на матрице
    x += lcd->x_offs;
    y += lcd->y_offs;
    x1 += lcd->x_offs;
    y1 += lcd->y_offs;
    //Создаем управляющую строку для интерпретатора драйвера дисплея, которая укажет контроллеру дисплея,
    //что мы определяем область памяти и хотим прочитать эту область. Предварительно переведем дисплей в
    //режим цвета 18 бит, так как команда Memory Read (0x2E) должна работать только с режимом цвета 18 бит.
    uint8_t set_win_and_read[] = { //Команда выбора режима цвета
                                   LCD_UPR_COMMAND, 0x3A, 1, 0x66, //0x66 - 18-битный цвет, 0x55 - 16-битный
                                   //Установка адресов, определяющих блок:
                                   LCD_UPR_COMMAND, 0x2A, 4, 0, 0, 0, 0, //столбец
                                   LCD_UPR_COMMAND, 0x2B, 4, 0, 0, 0, 0, //строка
				   //Команда Memory Read (читать память)
				   LCD_UPR_COMMAND, 0x2E, 0,
				   LCD_UPR_END	};
    //Вписываем в управляющую строку координаты заданного окна
    set_win_and_read[7]  = x >> 8;  set_win_and_read[8]  = x & 0xFF;
    set_win_and_read[9]  = x1 >> 8; set_win_and_read[10]  = x1 & 0xFF;
    set_win_and_read[14] = y >> 8;  set_win_and_read[15] = y & 0xFF;
    set_win_and_read[16] = y1 >> 8; set_win_and_read[17] = y1 & 0xFF;
    //Ждем, когда дисплей освободится и будет готов к приему новых команд и данных
    while (LCD_GetState(lcd) != LCD_STATE_READY) { __NOP(); }
    lcd->cs_control = 1; //Отключаем управление линией CS со стороны интерпретатора управляющих строк
    SPI_TypeDef *spi = lcd->spi_data.spi;
    uint32_t spi_param = spi->CR1; //Запоминаем параметры spi
    //Настраиваем spi (spi у нас уже выключен)
    spi->CR1 &= ~(SPI_CR1_BIDIMODE | //4-x проводный spi
                  SPI_CR1_RXONLY   |
                  SPI_CR1_CRCEN    |
                  SPI_CR1_BR_Msk   | //Маска скорости spi
                  SPI_CR1_DFF);      //Ширина кадра 8 бит
    //Установим скорость spi для чтения дисплея.
    //Параметр SPI_SPEED_DISPLAY_READ настраивается в display.h
    //Дело в том, что согласно спецификаций на контроллеры дисплея, скорость
    //в режиме чтения из контроллера, как правило, ниже скорости в режиме записи данных
    //в контроллер.
    spi->CR1 |= (uint32_t)((SPI_SPEED_DISPLAY_READ & 7) << 3);
    LCD_CS_LOW //Подключаем контроллер дисплея к МК
    //Отправляем через интерпретатор управляющую строку на контроллер дисплея
    //"Дергать" выводом CS после отправки команды 0x2E нельзя, т.к. контроллер
    //дисплея может "подумать", что мы хотим прерывать чтение.
    LCD_String_Interpretator(lcd, set_win_and_read);
    LCD_DC_HI //Вывод DC контроллера дисплея установим в положении "данные". Но, мои эксперименты показывают,
    //что чтение работает и в положении "команда", что странно, т.к. согласно спецификации, первая команда, в т.ч.,
    //NOP должна прерывать операцию чтения памяти контроллера. В общем, чтение идет до тех пор, пока мы не прочитаем
    //всю выбранную нами область, либо пока тупо не прервем процесс чтения.
    uint32_t len = w * h; //Количество пикселей для чтения
    uint16_t *data_ptr = data; //Указатель на местоположение буфера для хранения пикселей
    uint8_t r, g, b; //Переменные с цветовыми составляющими
    spi->CR1 |= SPI_CR1_SPE; //Включаем spi
    //-------------------------- 8 холостых тактов (читай спецификацию) --------------------------
    //Ага... 8! На самом деле 16. Мать его 16!!! И это мне вынесло мозг, но догадался потому, что
    //смещены цвета были и последний пиксель в массиве имел "другой" цвет.
    //Контроллер дисплея за эти 8 тактов подготавливается к передаче данных о цвете пикселей
    int i = 2;
    while (i--) {
        spi->DR = 0x00; //NOP
	while (!(spi->SR & SPI_SR_RXNE)) { __NOP(); } //Ожидаем прием ответа от контроллера дисплея
	r = spi->DR;
    }
    //------------------------------ Читаем данные о цвете len пикселей --------------------------
    while (len--) {
        //Считываем последовательно цветовые составляющие
        //По спецификации последовательность считываемых составляющих цветов заявлена r, g, b,
        //Если считываемые цвета будут не соответствовать фактическим, то снизьте скорость spi для чтения,
        //но иногда стабильности чтения помогает подтяжка к питанию линии MISO spi.
        spi->DR = 0x00; //NOP
	while (!(spi->SR & SPI_SR_RXNE)) { __NOP(); }//Ожидаем прием ответа от контроллера дисплея
	r = spi->DR;
	spi->DR = 0x00; //NOP
	while (!(spi->SR & SPI_SR_RXNE)) { __NOP(); }
	g = spi->DR;
	spi->DR = 0x00; //NOP
	while (!(spi->SR & SPI_SR_RXNE)) { __NOP(); }
	b = spi->DR;
	*data_ptr++ = LCD_Color(lcd, r, g, b); //Преобразуем цвет из R8G8B8 в R5G6B5 и запоминаем его
    }
    LCD_CS_HI //Отключаем контроллер дисплея от МК
    lcd->cs_control = 0; //Включаем управление линией CS со стороны интерпретатора управляющих строк
    while (spi->SR & SPI_SR_BSY) { __NOP(); } //Ждем когда spi освободится
    spi->CR1 &= ~SPI_CR1_SPE; //spi выключаем
    spi->CR1 = spi_param; //Восстанавливаем прежние параметры spi
    //Восстанавливаем 16-битный режим цвета
    uint8_t color_restore[] = { LCD_UPR_COMMAND, 0x3A, 1, 0x55, LCD_UPR_END };
    LCD_String_Interpretator(lcd, color_restore);
}
```

Автор: **VadRov**\
Контакты: [Youtube](https://www.youtube.com/@VadRov) [Дзен](https://dzen.ru/vadrov) [VK](https://vk.com/vadrov) [Telegram](https://t.me/vadrov_channel)\
Донат: [Поддержать автора](https://yoomoney.ru/to/4100117522443917)
