# Isolamento de Trafego e Mitigacao de Conflito de Servidores DHCP

## Estudo de caso

Resolucao de um incidente de concorrencia de Camada 2 entre a Prefeitura e a Secretaria de Saude.

Este repositorio documenta o diagnostico, a analise de pacotes e a solucao de engenharia de redes aplicada para resolver um incidente grave de infraestrutura e instabilidade.

## 1. Visao geral do caso

O problema ocorria no ambiente da Secretaria de Saude, que estava interconectado ao predio principal da Prefeitura.

Devido a um erro de montagem e entrega de infraestrutura por parte da operadora de telecomunicacoes, os dominios de broadcast dos dois orgaos publicos ficaram unificados indevidamente na Camada 2. Isso gerava vazamento de requisicoes DHCP, atribuicoes incorretas de enderecamento IP, conflitos e quedas constantes de conectividade em computadores, impressoras e dispositivos moveis da unidade de saude.

## 2. Cenario e problema

### Infraestrutura de redes envolvida

| Componente | Endereco ou finalidade |
| --- | --- |
| Servidor DHCP da Prefeitura | `172.168.103.1` |
| Servidor DHCP da Secretaria de Saude | `192.168.1.254` |
| Sub-rede implementada para isolamento | `192.168.50.0/24` |

### Impactos e sintomas observados

- **Conflito de escopos:** sempre que ocorria uma solicitacao de renovacao ou concessao de lease (`DHCP Discover`/`DHCP Request`), estacoes de trabalho, celulares e impressoras da Secretaria de Saude capturavam enderecos IP pertencentes a rede da Prefeitura (`172.168.103.x`). Da mesma forma, dispositivos da Prefeitura capturavam IPs da Saude (`192.168.1.x`).
- **Indisponibilidade de servicos:** com a troca indevida de gateway e mascara de sub-rede, impressoras de rede e sistemas locais de atendimento da Saude ficavam totalmente inacessiveis.
- **Causa raiz:** falha de isolamento de enlace no link da operadora, resultando em uma disputa de corrida (*race condition*) entre dois servidores DHCP operando no mesmo dominio de broadcast fisico.

### Diagrama do problema: vazamento em Camada 2

```text
[ PREDIO DA PREFEITURA ]                   [ SECRETARIA DE SAUDE ]
  DHCP: 172.168.103.1                        DHCP: 192.168.1.254
          |                                          |
          +------------------+-----------------------+
                             |
                 [ VAZAMENTO DE BROADCAST ]
              (Falha na entrega do link da operadora)
                             |
              [ Dispositivos da Saude recebendo ]
              [   IPs do escopo da Prefeitura    ]
```

## 3. Analise de trafego e diagnostico via CLI

Para diagnosticar a causa exata sem interromper o trafego em producao, foram combinadas ferramentas de analise passiva de quadros (`tcpdump`) e reconhecimento ativo de servicos (`nmap`).

### 3.1 Mapeamento de servidores DHCP e portas com Nmap

Utilizou-se o Nmap para varrer e mapear ativamente os servicos DHCP visiveis no segmento e validar a porta UDP 67 aberta na rede:

```bash
sudo nmap -sU -p 67 --script broadcast-dhcp-discover
```

### 3.2 Captura e inspecao de pacotes em tempo real com tcpdump
Para acompanhar a troca de mensagens do fluxo DORA e identificar o vazamento das respostas:

```bash
sudo tcpdump -i eth0 -n "port 67 or port 68" -v
```

### Evidencias coletadas no trafego

A analise revelou que, logo apos o envio do quadro DHCP Discover (`255.255.255.255`) por um cliente da Secretaria de Saude, a interface recebia pacotes DHCP Offer originados do IP `172.168.103.1:67` (Prefeitura) quase simultaneamente aos pacotes do servidor local `192.168.1.254:67`.

O servidor que respondesse com menor latencia entregava os parametros de rede ao cliente, confirmando a fusao indevida da Camada 2.

## 4. Solucao aplicada e nova topologia

Como a adequacao do enlace por parte da operadora demandaria tempo, foi projetada e implementada uma solucao local de arquitetura de rede utilizando um roteador de fronteira para isolar o trafego na Camada 3.

### Nova arquitetura fisica e logica da Secretaria de Saude

```text
                    LINK DA OPERADORA / PREFEITURA
                                  |
                         MODEM DA OPERADORA
                                  |
                    (Enlace fisico RJ45 / uplink)
                                  |
             +------------------------------------------+
             |       ROTEADOR IMPLEMENTADO             |
             |             (FRONTEIRA)                  |
             |                                          |
             |  Interface WAN: cliente DHCP             |
             |  Interface LAN: 192.168.50.1 / gateway   |
             |  Servidor DHCP nativo habilitado         |
             +----------------------+-------------------+
                                    |
                         Cabo tronco / uplink
                                    |
             +------------------------------------------+
             |       SWITCHES NAO GERENCIAVEIS         |
             |              (Cascateamento L2)          |
             +----------------------+-------------------+
                                    |
              +---------------------+---------------------+
              |                     |                     |
          Computadores          Impressoras          Dispositivos
           de setor             de rede                  moveis
         (192.168.50.x)       (192.168.50.x)          (192.168.50.x)
```

### Passos da implementacao

1. **Criacao da sub-rede isolada:** definicao do novo escopo exclusivo `192.168.50.0/24` para atender todos os hosts da Secretaria de Saude.
2. **Instalacao e configuracao do roteador de fronteira:**
   - **Interface WAN (cliente DHCP):** conectada ao modem da operadora, obtendo automaticamente o IP de borda e isolando as requisicoes internas.
   - **Interface LAN (`192.168.50.1`):** conectada aos switches nao gerenciaveis da rede interna, servindo como novo gateway e servidor DHCP primario.
3. **Bloqueio de broadcast (contencao de Camada 3):** roteadores nao repassam pacotes de broadcast por padrao. As requisicoes DHCP Discover enviadas em broadcast (`255.255.255.255`) pelos clientes conectados aos switches nao gerenciaveis ficaram estritamente contidas na LAN `192.168.50.x`, impedindo o vazamento para o modem e a rede da Prefeitura (`172.168.103.1`).

## 5. Resultados alcancados

| Metrica / objetivo | Status anterior | Status atual (pos-mitigacao) |
| --- | --- | --- |
| Isolamento DHCP | Concorrencia de escopos entre Prefeitura e Saude | 100% isolado na sub-rede `192.168.50.x` |
| Estabilidade de hosts | Quedas constantes e troca aleatoria de gateway | Conexao estavel e leases controlados |
| Perifericos de rede | Impressoras inacessiveis por troca de IP | Acessibilidade continua mantida |

## 6. Estrutura do repositorio

```text
.
├── captures/
│   └── dhcp_leak_capture.pcap       # Captura de pacotes sanitizada do incidente
├── docs/
│   └── network_topology.png          # Diagrama visual do ambiente
└── readme.md                         # Documentacao tecnica do repositorio
```

## 7. Autor

**Gabriel de Souza do Nascimento**  
Analista de Suporte N2 / Infraestrutura de TI

- GitHub: [github.com/GabrielNascimentoTI](https://github.com/GabrielNascimentoTI)
- LinkedIn: [linkedin.com/in/gabrielnascimentosouza](https://linkedin.com/in/gabrielnascimentosouza)
