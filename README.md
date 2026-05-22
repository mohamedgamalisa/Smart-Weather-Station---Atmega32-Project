# Smart-Weather-Station---Atmega32-Project
#define F_CPU 1000000UL

#include <avr/io.h>
#include <util/delay.h>
#include <stdio.h>
#include <stdlib.h>



// ================= LCD I2C ADDRESS =================
#define LCD_ADDR 0x27



// ================= TWI FUNCTIONS =================
void TWI_Init()
{
    TWBR = 2;
}

void TWI_Start()
{
    TWCR = (1<<TWINT)|(1<<TWSTA)|(1<<TWEN);

    while (!(TWCR & (1<<TWINT)));
}

void TWI_Stop()
{
    TWCR = (1<<TWINT)|(1<<TWEN)|(1<<TWSTO);
}

void TWI_Write(unsigned char data)
{
    TWDR = data;

    TWCR = (1<<TWINT)|(1<<TWEN);

    while (!(TWCR & (1<<TWINT)));
}



// ================= LCD FUNCTIONS =================
void LCD_Send_Command(char cmd)
{
    char cmd_u, cmd_l;

    cmd_u = (cmd & 0xF0);
    cmd_l = ((cmd << 4) & 0xF0);

    TWI_Start();
    TWI_Write(LCD_ADDR << 1);

    TWI_Write(cmd_u | 0x0C);
    TWI_Write(cmd_u | 0x08);

    TWI_Write(cmd_l | 0x0C);
    TWI_Write(cmd_l | 0x08);

    TWI_Stop();
}

void LCD_Send_Data(char data)
{
    char data_u, data_l;

    data_u = (data & 0xF0);
    data_l = ((data << 4) & 0xF0);

    TWI_Start();
    TWI_Write(LCD_ADDR << 1);

    TWI_Write(data_u | 0x0D);
    TWI_Write(data_u | 0x09);

    TWI_Write(data_l | 0x0D);
    TWI_Write(data_l | 0x09);

    TWI_Stop();
}

void LCD_Init()
{
    _delay_ms(50);

    LCD_Send_Command(0x02);
    LCD_Send_Command(0x28);
    LCD_Send_Command(0x0C);
    LCD_Send_Command(0x06);
    LCD_Send_Command(0x01);

    _delay_ms(2);
}

void LCD_Clear()
{
    LCD_Send_Command(0x01);

    _delay_ms(2);
}

void LCD_Set_Cursor(uint8_t row, uint8_t col)
{
    uint8_t pos;

    if(row == 0)
        pos = 0x80 + col;
    else
        pos = 0xC0 + col;

    LCD_Send_Command(pos);
}

void LCD_String(char *str)
{
    while(*str)
    {
        LCD_Send_Data(*str++);
    }
}



// ================= ADC =================
void ADC_Init()
{
    ADMUX = (1<<REFS0);

    ADCSRA =
    (1<<ADEN)  |
    (1<<ADPS2) |
    (1<<ADPS1);
}

uint16_t ADC_Read(uint8_t channel)
{
    channel &= 0x07;

    ADMUX = (ADMUX & 0xF8) | channel;

    ADCSRA |= (1<<ADSC);

    while(ADCSRA & (1<<ADSC));

    return ADCW;
}



// ================= DHT11 =================
#define DHT_DDR DDRD
#define DHT_PORT PORTD
#define DHT_PIN PIND
#define DHT_INPUT PD6

uint8_t Rh_byte1, Rh_byte2;
uint8_t Temp_byte1, Temp_byte2;
uint8_t checksum;

void Request()
{
    DHT_DDR |= (1<<DHT_INPUT);

    DHT_PORT &= ~(1<<DHT_INPUT);

    _delay_ms(20);

    DHT_PORT |= (1<<DHT_INPUT);

    _delay_us(30);
}

void Response()
{
    DHT_DDR &= ~(1<<DHT_INPUT);

    while(DHT_PIN & (1<<DHT_INPUT));
    while(!(DHT_PIN & (1<<DHT_INPUT)));
    while(DHT_PIN & (1<<DHT_INPUT));
}

uint8_t Receive_data()
{
    uint8_t c = 0;

    for(uint8_t q=0; q<8; q++)
    {
        while(!(DHT_PIN & (1<<DHT_INPUT)));

        _delay_us(30);

        if(DHT_PIN & (1<<DHT_INPUT))
            c = (c<<1) | 1;
        else
            c = (c<<1);

        while(DHT_PIN & (1<<DHT_INPUT));
    }

    return c;
}



// ================= RGB =================
void RGB_Init()
{
    DDRB |= (1<<PB0) | (1<<PB1) | (1<<PB2);
}

void RGB_Red()
{
    PORTB = (1<<PB0);
}

void RGB_Green()
{
    PORTB = (1<<PB1);
}

void RGB_Blue()
{
    PORTB = (1<<PB2);
}



// ================= MAIN =================
int main(void)
{
    char line1[17];
    char line2[17];
    char direction[10];

    int windSpeed;

    uint8_t color = 0;

    TWI_Init();
    LCD_Init();
    ADC_Init();
    RGB_Init();

    LCD_Clear();

    while(1)
    {
        // ===== DHT11 =====
        Request();
        Response();

        Rh_byte1   = Receive_data();
        Rh_byte2   = Receive_data();
        Temp_byte1 = Receive_data();
        Temp_byte2 = Receive_data();
        checksum   = Receive_data();



        // ===== WIND SPEED =====
        uint16_t windADC = ADC_Read(0);


        // Ignore noise when motor is still
        if(windADC < 60)
        {
            windSpeed = 0;
        }
        else
        {
            windSpeed =
            ((windADC - 60) * 30UL) / 963;
        }



        // ===== WIND DIRECTION =====
        uint16_t dirADC = ADC_Read(1);

        if(dirADC < 200)
            sprintf(direction,"North");

        else if(dirADC < 400)
            sprintf(direction,"East");

        else if(dirADC < 700)
            sprintf(direction,"South");

        else
            sprintf(direction,"West");



        // ===== LCD DISPLAY =====
        LCD_Clear();



        // Line 1
        LCD_Set_Cursor(0,0);

        sprintf(line1,
        "T:%dC H:%d%%",
        Temp_byte1,
        Rh_byte1);

        LCD_String(line1);



        // Line 2
        LCD_Set_Cursor(1,0);

        sprintf(line2,
        "Speed:%dm/s",
        windSpeed);

        LCD_String(line2);



        // Direction
        LCD_Set_Cursor(1,11);

        LCD_String(direction);



        // ===== RGB EFFECT =====
        if(color == 0)
        {
            RGB_Red();
        }
        else if(color == 1)
        {
            RGB_Green();
        }
        else
        {
            RGB_Blue();
        }

        color++;

        if(color > 2)
            color = 0;



        _delay_ms(1000);
    }
}
