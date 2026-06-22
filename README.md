# Atmega328p-robot-vaccum--cleaner
/* 
The candidate should design an application capable of simulating the control of a
robot vacuum cleaner that is turned on by a predefined sequence of
buttons located on the robot itself.
This startup sequence is simulated by pressing the following
buttons in succession: B2, B3, B2, B3, B2.
Pressing the buttons causes, in any case,
LEDs L0, L1, and L2 to light up
in the following order (see Table):
If, at any point in the sequence,
no button is pressed for more than 5 seconds,
the stored button sequence must be reset.
Once all 5 buttons have been pressed, the
system verifies that the combination is correct. If
the sequence is correct, LEDs L0 through L2 remain in their current state, and
LED L3 lights up, indicating a correct sequence. Only in this
case is the robot also activated; it begins vacuuming by turning on
its motor and activating the wheels for movement. If the button sequence is
incorrect, all LEDs must turn off, and the system must restart from its
initial state.
The robot’s operating sequence consists of the following phases:
• Vacuuming (duration: 4 seconds), during which LED L5 flashes with a period of 300 ms and
a duty cycle of 33%;
• Movement (duration: 2 seconds), during which LED L4 flashes with a period of 500 ms
and a duty cycle of 20%.
The robot begins alternating between these two phases until it is stopped by
pressing button B5, at which point the LEDs turn off.
*/



#include <avr/io.h>
#include <avr/interrupt.h>

#define B2 (1<<PIND2)
#define B3 (1<<PIND3)
#define B4 (1<<PIND4)
#define B5 (1<<PIND5)
#define B6 (1<<PIND6)
#define B7 (1<<PIND7)

typedef enum {PRIMO, SECONDO, TERZO, QUARTO, QUINTO, VERIFICA, ASPIRAZIONE, SPOSTAMENTO}states_t;
	states_t statocorrente = PRIMO;
	
volatile uint16_t pressed;
volatile uint16_t old= 0xff;

volatile uint16_t tick;
volatile uint16_t ciclo;
volatile uint16_t timer;

volatile uint16_t vero=0;
volatile uint16_t pressioneT=0x00;



int main(void)
{
	DDRC |= 0x3f;
	PORTC &=~ 0x3f;
	
	DDRD &=~ 0xfc;
	PORTD |= 0xfc;
	
	PCICR |= (1<<PCIE2);
	PCMSK2 |= 0xfc;
	
	TCCR0A |= (1<<WGM01);
	TIMSK0 |= (1<<OCIE0A);
	OCR0A =77; //--->5ms 
	
	TCCR1B |= (1<<WGM12);
	TIMSK1 |= (1<<OCIE1A);
	OCR1A =7812; //--->0,5s
	
	/*attivazione timer
	TCCR0B |=(1<<CS00)|(1<<CS02);
	TCCR1B |=(1<<CS10)|(1<<CS12);
	*/
	sei();
	
    while(1)
    {
        switch(statocorrente)
		{
			case PRIMO:
			if((pressed==B2)||(pressed==B3)||(pressed==B4)||(pressed==B5)||(pressed==B6)||(pressed==B7))
			{
				pressioneT=0x1;
				PORTC = pressioneT;
				if(pressed== B2)
				{
					vero++;
					pressed=0;
				}
				pressed=0;
				statocorrente= SECONDO;
				TCCR1B |=(1<<CS10)|(1<<CS12);
			}
			
			
			break;
			
			
			
			case SECONDO:
			if((pressed==B2)||(pressed==B3)||(pressed==B4)||(pressed==B5)||(pressed==B6)||(pressed==B7))
			{
				pressioneT=0x2;
				PORTC = pressioneT;
				if(pressed== B3)
				{
					vero++;
					pressed=0;
				}
				pressed=0;
				statocorrente= TERZO;
			}
			if(ciclo==10)
			{
				pressed=0;
				statocorrente= PRIMO;
				PORTC = (0x0);
				ciclo=0;
				vero=0;
			}
			
			
			break;
			
			
			
			case TERZO:
			if((pressed==B2)||(pressed==B3)||(pressed==B4)||(pressed==B5)||(pressed==B6)||(pressed==B7))
			{
				pressioneT=0x3;
				PORTC = pressioneT;
				if(pressed== B2)
				{
					vero++;
					pressed=0;
				}
				pressed=0;
				statocorrente= QUARTO;
			}
			if(ciclo==10)
			{
				pressed=0;
				statocorrente= PRIMO;
				PORTC = (0x0);
				ciclo=0;
				vero=0;
			}
			
			break;
			
			
			case QUARTO:
			if((pressed==B2)||(pressed==B3)||(pressed==B4)||(pressed==B5)||(pressed==B6)||(pressed==B7))
			{
				pressioneT=0x4;
				PORTC = pressioneT;
				if(pressed== B3)
				{
					vero++;
					pressed=0;
				}
				pressed=0;
				statocorrente= QUINTO;
			}
			if(ciclo==10)
			{
				pressed=0;
				statocorrente= PRIMO;
				PORTC = (0x0);
				ciclo=0;
				vero=0;
			}
			
			break;
			
			
			case QUINTO:
			if((pressed==B2)||(pressed==B3)||(pressed==B4)||(pressed==B5)||(pressed==B6)||(pressed==B7))
			{
				pressioneT=0x5;
				PORTC = pressioneT;
				if(pressed== B2)
				{
					vero++;
					pressed=0;
				}
				pressed=0;
				statocorrente= VERIFICA;
			}
			if(ciclo==10)
			{
				pressed=0;
				statocorrente= PRIMO;
				PORTC = (0x0);
				ciclo=0;
				vero=0;
				
			}
			
			break;
			
			
			case VERIFICA:
			if(vero==5)
			{
				vero=0;
				ciclo=0;
				tick=0;
				TCCR0B |=(1<<CS00)|(1<<CS02);
				PORTC |= (1<<PORTC3);
				statocorrente= ASPIRAZIONE;
			}
			else 
			{
				statocorrente = PRIMO;
				PORTC = 0x0;
				tick=0;
				ciclo=0;
				pressed=0;
				vero=0;
				
		
			}
			break;
			
			
			case ASPIRAZIONE:
			if (ciclo==8)
			{
				tick=0;
				ciclo=0;
				pressed=0;
				vero=0;
				statocorrente= SPOSTAMENTO;
				PORTC &=~ (1<<PORTC5);
				PORTC |= (1<<PORTC4);
			}
			if(pressed==B5)
			{
				statocorrente= PRIMO;
				tick=0;
				ciclo=0;
				pressed=0;
				vero=0;
				PORTC= 0x0;
				TCCR0B &=~ (1<<CS00)|(1<<CS02);
				TCCR1B &=~ (1<<CS10)|(1<<CS12);
				
			}
			break;
			case SPOSTAMENTO:
			if (ciclo==4)
			{
				tick=0;
				ciclo=0;
				pressed=0;
				vero=0;
				statocorrente= ASPIRAZIONE;
				PORTC &=~ (1<<PORTC4);
				PORTC |= (1<<PORTC5);
			}
			if(pressed==B5)
			{
				statocorrente= PRIMO;
				tick=0;
				ciclo=0;
				pressed=0;
				vero=0;
				PORTC= 0x0;
				TCCR0B &=~ (1<<CS00)|(1<<CS02);
				TCCR1B &=~ (1<<CS10)|(1<<CS12);
				
			}
			break;
			
		}
		 
    }
}
ISR(TIMER1_COMPA_vect)
{
	ciclo++;

}
ISR(TIMER0_COMPA_vect)
{
	tick++;
	if (statocorrente==ASPIRAZIONE)
	{
		if (tick>=20)
		{
			PORTC &=~ (1<<PORTC5);
			
			
		}
		if (tick==60)
		{
			PORTC |= (1<<PORTC5);
			tick=0;
		}
	}
	if (statocorrente==SPOSTAMENTO)
	{
		if (tick>=20)
		{
			PORTC &=~ (1<<PORTC4);
		}
		if (tick==100)
		{
			PORTC |= (1<<PORTC4);
			tick=0;
		}
	}
}
ISR(PCINT2_vect)
{
	uint16_t cambiamento = old ^ PIND;
	old= PIND;
	
	if (cambiamento&(1<<PIND2))
	{
		if ((old&(1<<PIND2))==0)
		{
			pressed= (1<<PIND2);
		}
	}
	if (cambiamento&(1<<PIND3))
	{
		if ((old&(1<<PIND3))==0)
		{
			pressed= (1<<PIND3);
		}
	}
	if (cambiamento&(1<<PIND4))
	{
		if ((old&(1<<PIND4))==0)
		{
			pressed= (1<<PIND4);
		}
	}
	if (cambiamento&(1<<PIND5))
	{
		if ((old&(1<<PIND5))==0)
		{
			pressed= (1<<PIND5);
		}
	}
	if (cambiamento&(1<<PIND6))
	{
		if ((old&(1<<PIND6))==0)
		{
			pressed= (1<<PIND6);
		}
	}
	if (cambiamento&(1<<PIND7))
	{
		if ((old&(1<<PIND7))==0)
		{
			pressed= (1<<PIND7);
		}
	}

}
