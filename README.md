# Estacionamento Inteligente (ESP32 + MQTT)

Sistema de monitoramento de vagas de estacionamento. Cada vaga tem um sensor ultrassônico que detecta se há um carro, LEDs que indicam o estado no local, um display LCD e um painel web que mostra tudo em tempo real.

- **Simulação no Wokwi:** https://wokwi.com/projects/481519264850453505
- **Painel web:** `painel_estacionamento.html` (abrir no navegador)

**Grupo:** _(nomes dos integrantes)_
**Disciplina / Professor:** _(preencher)_

---

## Como funciona

```
[Sensor HC-SR04 x4] -> [ESP32] -> LEDs + LCD (no local)
                          |
                          | Wi-Fi + MQTT (JSON)
                          v
                  [Broker HiveMQ] -> [Painel web]
```

1. O ESP32 mede a distância de cada vaga com um sensor ultrassônico.
2. Se a distância for menor que **50 cm**, a vaga é considerada **ocupada**.
3. O ESP32 acende o LED verde (livre) ou vermelho (ocupada) de cada vaga e atualiza o LCD.
4. O estado completo é publicado em JSON via MQTT, quando algo muda e também a cada 5 segundos.
5. O painel web assina o tópico e mostra as vagas, a contagem de livres e a ocupação.

A vaga 1 é **PCD** e tem um LED azul fixo de identificação.

---

## Requisitos da atividade

| Requisito | Como foi atendido |
|---|---|
| **R1 - Percepção e ação** | 4 sensores ultrassônicos HC-SR04 (percepção) e LEDs verde/vermelho por vaga (atuadores), que reagem à leitura de distância. |
| **R2 - Decisão local** | A regra `distância < 50 cm = ocupada` roda no próprio ESP32 (função `loop()`). LEDs e LCD continuam funcionando mesmo sem Wi-Fi ou internet. |
| **R3 - Comunicação sem fio** | Wi-Fi + **MQTT** (ver justificativa abaixo). |
| **R4 - Payload estruturado** | JSON documentado na seção [Payload](#payload-mqtt), com exemplo real. |
| **R5 - Interface** | LCD 16x2 no local e painel web, sem necessidade do Monitor Serial. |

### Por que MQTT?

- É um protocolo leve, feito para dispositivos IoT com poucos recursos como o ESP32.
- Usa o modelo publicar/assinar: o ESP32 publica o estado uma vez e qualquer número de painéis pode receber, sem o ESP32 precisar conhecê-los.
- Diferente de um servidor HTTP no próprio ESP32, o painel não precisa saber o IP do dispositivo nem estar na mesma rede.
- Com a mensagem retida (`retain`), um painel que abre depois já recebe o último estado conhecido.
- Bluetooth teria alcance curto e exigiria o aparelho por perto, o que não faz sentido para um estacionamento monitorado à distância.

---

## Hardware (simulado no Wokwi)

- ESP32 DevKit C v4
- 4x sensor ultrassônico HC-SR04
- 4x LED verde + 4x LED vermelho (um par por vaga)
- 1x LED azul (identificação da vaga PCD)
- 1x LCD 16x2 com interface I2C (endereço `0x27`)

### Pinagem

| Vaga | Tipo | TRIG | ECHO | LED verde | LED vermelho |
|---|---|---|---|---|---|
| 1 | PCD | 5 | 18 | 2 | 4 |
| 2 | Comum | 17 | 16 | 19 | 23 |
| 3 | Comum | 27 | 14 | 12 | 13 |
| 4 | Comum | 25 | 26 | 32 | 33 |

LED azul PCD: pino 15. LCD: I2C (SDA 21, SCL 22).

---

## Payload MQTT

- **Broker:** `broker.hivemq.com` (público)
- **Porta do ESP32:** 1883 (TCP) | **Porta do painel web:** 8884 (WebSocket seguro)
- **Tópico:** `iot_estacionamento/vagas/status`
- **Direção:** ESP32 publica, painel assina
- **Frequência:** a cada mudança de estado de qualquer vaga e, no mínimo, a cada 5 s
- **Retain:** sim

### Exemplo real

Vaga 2 ocupada, demais livres:

```json
{
  "total_vagas": 4,
  "vagas_livres": 3,
  "pcd_livre": true,
  "ocupacao_pct": 25,
  "vagas": [
    { "id": 1, "tipo": "PCD",   "status": "LIVRE",   "dist_cm": 400.0 },
    { "id": 2, "tipo": "COMUM", "status": "OCUPADA", "dist_cm": 32.4 },
    { "id": 3, "tipo": "COMUM", "status": "LIVRE",   "dist_cm": 400.0 },
    { "id": 4, "tipo": "COMUM", "status": "LIVRE",   "dist_cm": 400.0 }
  ],
  "timestamp_ms": 128450
}
```

### Campos

| Campo | Tipo | Descrição |
|---|---|---|
| `total_vagas` | inteiro | Quantidade total de vagas (4). |
| `vagas_livres` | inteiro | Quantas vagas estão livres no momento. |
| `pcd_livre` | booleano | `true` se a vaga PCD está livre. |
| `ocupacao_pct` | inteiro | Percentual de ocupação (0 a 100). |
| `vagas` | lista | Um objeto por vaga (campos abaixo). |
| `vagas[].id` | inteiro | Número da vaga (1 a 4). |
| `vagas[].tipo` | texto | `"PCD"` ou `"COMUM"`. |
| `vagas[].status` | texto | `"LIVRE"` ou `"OCUPADA"`. |
| `vagas[].dist_cm` | número | Distância lida pelo sensor, em cm (400 = sem eco, nada à frente). |
| `timestamp_ms` | inteiro | Tempo desde que o ESP32 ligou, em milissegundos. |

---

## Como executar

### 1. Simulação (ESP32)

1. Abra o [projeto no Wokwi](https://wokwi.com/projects/481313349475281921).
2. Clique em **Play**.
3. Clique em um sensor ultrassônico e arraste o controle de distância para menos de 50 cm. O LED da vaga troca de verde para vermelho e o LCD atualiza.

### 2. Painel web

1. Baixe o arquivo `painel_estacionamento.html`.
2. Abra no navegador (duplo clique). É necessário ter internet.
3. Com a simulação do Wokwi rodando, use o modo **Ao vivo (MQTT)**.
4. O modo **Simulação** do painel não precisa do Wokwi: clique nas vagas para estacionar ou liberar um carro.

> O tópico MQTT é público. Se mais de um grupo usar o mesmo nome, os dados se misturam. Para evitar isso, altere o tópico no `sketch.ino` (`TOPIC_STATUS`) e no `painel_estacionamento.html` (`TOPIC`) para o mesmo texto exclusivo do grupo.

---

## Estrutura do repositório

```
.
├── sketch.ino                    # Firmware do ESP32
├── diagram.json                  # Circuito do Wokwi
├── libraries.txt                 # Bibliotecas usadas
├── painel_estacionamento.html    # Painel web
└── README.md
```

## Bibliotecas

- [PubSubClient](https://github.com/knolleary/pubsubclient) (cliente MQTT)
- [ArduinoJson](https://arduinojson.org/) (montagem do JSON)
- [LiquidCrystal_I2C](https://github.com/johnrickman/LiquidCrystal_I2C) (display LCD)
- [MQTT.js](https://github.com/mqttjs/MQTT.js) (cliente MQTT do painel web, carregado por CDN)

## Detalhes de implementação

- O buffer do MQTT foi aumentado para 512 bytes (`setBufferSize`), pois o JSON das 4 vagas passa do limite padrão de 256 bytes da biblioteca.
- A reconexão ao Wi-Fi e ao broker é feita sem travar o `loop()`, com intervalo entre tentativas, para que a leitura dos sensores e a atuação nos LEDs nunca parem por falta de rede.
