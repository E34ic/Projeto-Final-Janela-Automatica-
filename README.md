#include <Boards.h>
#include <Firmata.h>
#include <FirmataConstants.h>
#include <FirmataDefines.h>
#include <FirmataMarshaller.h>
#include <FirmataParser.h>
#include <Stepper.h>
#include <Wire.h>
#include "Arduino.h"
#include "SI114X.h"

// Inicialização do sensor e do motor de passo
SI114X sensorLuz = SI114X();
const int passosRev = 32; // Número de passos por revolução
Stepper motor(passosRev, 8, 10, 9, 11);
const int limiar = 270; // Valor de referência para ativar o motor
const unsigned long tempoEspera = 2000; // Tempo de espera para evitar movimento contínuo
unsigned long ultimoMov = 0; // Tempo da última ação do motor
bool estaAberto = false; // Estado do motor (aberto ou fechado)

// Parâmetros do filtro de média móvel
const int numAmostras = 5; // Número de amostras para a média móvel
int amostrasVis[numAmostras]; // Array para armazenar as amostras de visibilidade
int indiceAmostra = 0; // Índice atual da amostra
int somaVis = 0; // Soma das amostras de visibilidade
int mediaVis = 0; // Valor da média móvel
int faixa =300;

// Filtro passa-alta
float alfa = 0.5; // Fator de filtragem (ajustável entre 0 e 1)
int visAnterior = 0; // Última leitura do sensor
int visFiltradaPassaAlta = 0; // Saída do filtro passa-alta


void setup() {
    Serial.begin(115200);  // Inicializa a comunicação serial
    motor.setSpeed(500);   // Configura a velocidade do motor

    // Inicializa o sensor
    while (!sensorLuz.Begin()) {
        delay(1000);       // Aguarda até que o sensor esteja pronto
    }

    // Inicializa o array de amostras
    for (int i = 0; i < numAmostras; i++) {
        amostrasVis[i] = 0; // Define todas as amostras como zero
    }
}

void loop() {
    // Lê os valores do sensor
    int visivel = sensorLuz.ReadVisible();
    int infravermelho = sensorLuz.ReadIR();
    float ultravioleta = sensorLuz.ReadUV() / 100;

    // Atualiza a média móvel
    somaVis -= amostrasVis[indiceAmostra]; // Remove a amostra mais antiga da soma
    amostrasVis[indiceAmostra] = visivel; // Adiciona a nova amostra
    somaVis += visivel; // Adiciona a nova amostra à soma
    indiceAmostra = (indiceAmostra + 1) % numAmostras; // Atualiza o índice circular

  // Filtro passa-alta aplicado à saída da média móvel
    visFiltradaPassaAlta = alfa * (visFiltradaPassaAlta + mediaVis - visAnterior);
    visAnterior = mediaVis; // Atualiza o valor anterior para a próxima iteração

    int mediaVis = somaVis / numAmostras; // Calcula a média

    // Controle do motor com base na média da leitura do sensor
    if (mediaVis >= limiar && !estaAberto && (millis() - ultimoMov >= tempoEspera)) {
        motor.step(512); // Gira o motor 2048 passos para abrir
        estaAberto = true; // Atualiza o estado para aberto
        ultimoMov = millis(); // Atualiza o tempo da última ação
       // Serial.println("Janela aberto"); // Indicação do motor sendo aberto
        //delay(500); // Aguarda um tempo após o movimento
    } else if (mediaVis < limiar && estaAberto && (millis() - ultimoMov >= tempoEspera)) {
        motor.step(-512); // Gira o motor -2048 passos para voltar
        estaAberto = false; // Atualiza o estado para fechado
        ultimoMov = millis(); // Atualiza o tempo da última ação
       // Serial.println("Janela fechado"); // Indicação do motor sendo fechado
        //delay(500); // Aguarda um tempo após o movimento
    }

    
    // Envia os dados para o Serial Plotter no formato correto
    Serial.print(mediaVis);    // Envia a visibilidade média
    Serial.print(",");         // Separa os valores com vírgula
    Serial.print(faixa);
    Serial.print(","); 
    //Serial.print(infravermelho); // Envia o valor IR
    //Serial.print(",");         // Separa os valores com vírgula
    //Serial.print(ultravioleta);  // Envia o valor UV
    //Serial.print(",");         // Separa os valores com vírgula
    Serial.println(estaAberto ? 300 : 200); // Envia o estado do motor (1=aberto, 0=fechado)

    delay(100); // Atraso para limitar a frequência de leitura
}
