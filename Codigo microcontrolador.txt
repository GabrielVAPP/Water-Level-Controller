#include <xc.h>

#define _XTAL_FREQ 4000000       // Frequência do oscilador para 4 MHz
#pragma config FOSC = INTRC_NOCLKOUT
#pragma config WDTE = OFF
#pragma config PWRTE = OFF
#pragma config MCLRE = OFF
#pragma config CP = OFF
#pragma config CPD = OFF
#pragma config BOREN = OFF
#pragma config IESO = OFF
#pragma config FCMEN = OFF
#pragma config LVP = OFF

// Definições para sensores e atuadores
#define SENSOR1 PORTAbits.RA0
#define SENSOR2 PORTAbits.RA1
#define SENSOR3 PORTAbits.RA2
#define BOMBA PORTBbits.RB0
#define ALARME PORTBbits.RB1

// Função para debounce
unsigned char debounce(volatile unsigned char *sensorPin) {
    unsigned char leitura1 = *sensorPin;
    __delay_ms(20);  // Atraso de debounce
    unsigned char leitura2 = *sensorPin;

    // Retorna valor estável
    return (leitura1 == leitura2) ? leitura1 : !leitura1;
}

void main() {
    // Configuração de pinos analógicos para digitais
    ANSEL = 0x00;      // Desabilita entradas analógicas no PORTA e pinos inferiores do PORTB
    ANSELH = 0x00;     // Desabilita entradas analógicas nos pinos superiores do PORTB

    // Configuração de entradas e saídas
    TRISBbits.TRISB0 = 0; // Configura BOMBA como saída
    TRISBbits.TRISB1 = 0; // Configura ALARME como saída
    TRISAbits.TRISA0 = 1; // Configura SENSOR1 como entrada
    TRISAbits.TRISA1 = 1; // Configura SENSOR2 como entrada
    TRISAbits.TRISA2 = 1; // Configura SENSOR3 como entrada

    // Desabilitar os resistores de pull-up para todos os pinos
    OPTION_REG = 0b10000000;
    WPUB = 0x00;     // Desativa pull-ups em PORTB
    
    CM1CON0 = 0x00; // Desativa o Comparador 1
    CM2CON0 = 0x00; // Desativa o Comparador 2
    
    // Configuração da frequência do oscilador interno para 4 MHz
    OSCCONbits.IRCF = 0b110; // Frequência do oscilador interno: 4 MHz
    OSCCONbits.SCS = 1;      // Use o oscilador interno como fonte de clock principal

    // Inicializa saídas
    BOMBA = 0;
    ALARME = 0;

    // Variáveis para monitoramento do temporizador
    unsigned int bombaTimer = 0;       // Contador para o tempo da bomba
    unsigned char bombaMonitorando = 0;  // Flag para ativar/desativar o monitoramento

    while (1) {


        // Leituras com debounce usando variáveis auxiliares
        unsigned char sensor1 = SENSOR1;
        unsigned char sensor1State = debounce(&sensor1);

        unsigned char sensor2 = SENSOR2;
        unsigned char sensor2State = debounce(&sensor2);

        unsigned char sensor3 = SENSOR3;
        unsigned char sensor3State = debounce(&sensor3);

        // Lógica de controle
        if (ALARME == 1) {
            if (sensor3State == 1) {
                ALARME = 0;  // Desativa o alarme quando SENSOR3 é acionado
            }
        } else if (sensor2State == 0) {
            BOMBA = 0;
            ALARME = 1;  // Ativa o alarme se SENSOR2 for acionado
        } else if (ALARME == 0 && sensor1State == 0) {
            BOMBA = 1;  // Liga a bomba quando SENSOR1 está ativo
        }

        // Monitoramento do estado da bomba
        if (sensor1State == 1 && BOMBA == 1) {
            // Inicia o monitoramento se ainda não estiver monitorando
            if (!bombaMonitorando) {
                bombaMonitorando = 1;  // Ativa o monitoramento
                bombaTimer = 0;        // Reseta o temporizador
            }
        }

        // Durante o monitoramento
        if (bombaMonitorando) {
            bombaTimer += 50;  // Incrementa o temporizador com o valor do atraso (50 ms)

            // Reavalie o estado do SENSOR2
            if (sensor2State == 0) {
                ALARME = 1;  // Ativa o alarme
                BOMBA = 0;   // Desliga a bomba
                bombaMonitorando = 0;  // Para o monitoramento
            } else if (bombaTimer >= 5000) {  // Se 10 segundos se passaram
                BOMBA = 0;  // Desliga a bomba
                bombaMonitorando = 0;  // Para o monitoramento
            }
        }
    }
}