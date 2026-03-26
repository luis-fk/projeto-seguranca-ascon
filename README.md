# Prova de Conceito: Criptografia Leve (Ascon-128) via ESP-NOW

Este repositório contém a implementação de um sistema de comunicação segura para dispositivos de Internet das Coisas (IoT) com recursos limitados. O projeto utiliza o padrão de criptografia leve **Ascon-128** (AEAD) sobre o protocolo **ESP-NOW**, garantindo confidencialidade, integridade e autenticidade na transmissão de dados.

A arquitetura foi desenhada com foco em segurança de hardware, implementando a geração de chaves via DRBG (Deterministic Random Bit Generator), armazenamento não-volátil (NVS) e mitigação de ataques de repetição utilizando nonces dinâmicos.

## 🛠️ Materiais Necessários
* **Hardware:** 2x Placas ESP32 (Sensor e Gateway) + Cabos Micro-USB.
* **Software:** IDE do Arduino (com o core do ESP32 instalado).
* **Dependência:** Biblioteca [ascon-suite](https://github.com/rweather/ascon-suite) instalada na IDE do Arduino.

---

## Passo 1: Geração da Chave Criptográfica Segura
A primeira etapa é gerar uma chave simétrica de 128 bits (16 bytes) utilizando entropia do sistema operacional. O repositório oferece duas abordagens:

### Opção A: O Método Rápido (Python)
Ideal para testes rápidos. O script solicita entropia diretamente ao Kernel do sistema.
1. Tenha o Python 3 instalado.
2. No terminal, execute:
   ```bash
   python3 gerador.py
   ```
3. Copie o *array* em C gerado no terminal.

### Opção B: O Método em C + mbedTLS
Demonstra o fluxo criptográfico de forma mais completa.
1. No Ubuntu/Linux, instale as bibliotecas de desenvolvimento:
   ```bash
   sudo apt update && sudo apt install libmbedtls-dev
   ```
2. Compile e execute o código:
   ```bash
   gcc gerador.c -o gerador -lmbedcrypto
   ./gerador
   ```
3. Copie o *array* em C gerado no terminal.

---

## Passo 2: Provisionamento Seguro (NVS)
A chave gerada **nunca** deve ser fixada (*hardcoded*) no código final. Vamos gravá-la na memória Flash (NVS) das duas placas ESP32.

1. Abra o arquivo `gravador.cpp` (ou `.ino`) na IDE do Arduino.
2. Localize a variável `key_to_save` e cole os 16 bytes que você gerou no Passo 1.
3. Conecte o **primeiro ESP32**, selecione a porta e faça o upload.
4. Abra o **Monitor Serial** (115200 baud). Você deverá ver a mensagem: `"SUCESSO! A chave de 128 bits foi salva permanentemente na NVS."`
5. Conecte o **segundo ESP32** e repita o processo de upload.

Agora, ambas as placas possuem a mesma chave armazenada de forma segura na memória não-volátil.

---

## Passo 3: Configurando o Gateway (Receptor)
Com as chaves gravadas, vamos preparar o nó que ficará "escutando" a rede.

1. Conecte o ESP32 que atuará como Gateway.
2. Abra o arquivo `receptor.cpp` (ou `.ino`) na IDE.
3. Faça o upload do código. 
4. Abra o Monitor Serial. Se tudo estiver correto, você verá a mensagem: `"Gateway pronto e escutando a rede ESP-NOW aguardando sensores..."`.
5. Deixe esta placa conectada para monitorar a chegada dos pacotes.

---

## Passo 4: Ligando o Sensor (Transmissor)
Por fim, vamos acionar o nó que fará a leitura, a criptografia e a transmissão.

1. Conecte o outro ESP32.
2. Abra o arquivo `transmissor.cpp` (ou `.ino`) na IDE e faça o upload.
3. Este código faz três coisas importantes a cada ciclo de 5 segundos:
   * **Lê a NVS** para recuperar a chave mestre.
   * **Incrementa o Nonce** matematicamente para garantir que a assinatura criptográfica seja sempre única, prevenindo ataques de *Replay*.
   * **Criptografa e Transmite** o pacote contendo `[Nonce] + [Texto Cifrado + Tag de Autenticação]` para o endereço de *broadcast* do ESP-NOW.

**O Resultado:**
Se você observar o Monitor Serial da placa **Receptora** (Gateway), verá os pacotes chegando, o Nonce sendo extraído, a assinatura Ascon sendo validada e a mensagem original sendo impressa na tela com sucesso a cada 5 segundos.
