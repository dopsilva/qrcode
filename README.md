# Sea QRCode

Biblioteca completa de **leitura e escrita de QR Codes (ISO/IEC 18004)** escrita em **[Sea](https://github.com/dopsilva/sea)** (<https://github.com/dopsilva/sea>), utilizando `libs/compress` para leitura e escrita nativa de imagens PNG.

## Recursos

- **100% em [Sea](https://github.com/dopsilva/sea)**: implementação completa do padrão ISO/IEC 18004 (Versões **1 a 40**, tamanhos de `21×21` até `177×177` módulos) sem dependências externas.
- **Correção de erros Reed-Solomon e BCH em $\text{GF}(2^8)$**:
  - Suporte aos 4 níveis de correção: **L** (~7%), **M** (~15%), **Q** (~25%) e **H** (~30%)
  - Codificador polinomial e decodificador completo (**Berlekamp-Massey**, busca de **Chien** e algoritmo de **Forney**) capaz de recuperar QR Codes parcialmente danificados, rasurados ou com logotipo central
  - Decodificação BCH(15, 5) para Format Info e BCH(18, 6) para Version Info por distância de Hamming (tolerando até 3 bits corrompidos)
- **Modos de codificação**:
  - Detecção automática ou explícita de modo **Numérico** (`0-9`), **Alfanumérico** (tabela de 45 caracteres), **Byte / UTF-8** (acentos, emojis e dados binários), **Kanji** (leitura Shift-JIS), **ECI** e **Multi-segmento**
  - Seleção automática da menor versão (`1..40`) e avaliação das **8 máscaras padrão (`0..7`)** pelas 4 regras de penalidade ISO/IEC 18004 (`N1..N4`)
- **Formatos de leitura e escrita suportados**:
  - **PNG** (1-bit e 8-bit Grayscale, RGB, RGBA, Palette + filtros Paeth, Sub, Up, Average)
  - **SVG** (escrita vetorial compacta `<path>` e leitura direta)
  - **BMP** (1, 8, 24 e 32 bits)
  - **PGM / PPM / PBM** (`P1` a `P6`)
  - **Terminal Unicode** (`▀`, `▄`, `█`, ` `) e **ASCII Art**
- **Detector de imagem robusto**:
  - Binarização global de **Otsu**, binarização **adaptativa** via imagem integral e detecção automática de QR Codes **invertidos** (claro sobre fundo escuro) e **rotacionados** (`0°`, `90°`, `180°`, `270°`)
  - Scanner de **Finder Patterns (`1:1:3:1:1`)** horizontal, vertical e diagonal + localização de **Alignment Pattern (`1:1:1`)** e amostragem de grade com correção de perspectiva/homografia
- **Payloads estruturados prontos para uso (`qrcode.payload`)**:
  - **PIX (BR Code EMV® QRCPS-MPM + CRC16-CCITT `0x1021`)**: geração e parser/validador de chaves, valores, TXID e CRC16
  - **Wi-Fi** (`WIFI:T:WPA;S:...;P:...;;`)
  - **vCard 3.0** (`BEGIN:VCARD ... END:VCARD`)
  - **TOTP / 2FA** (`otpauth://totp/...`), **Mailto**, **SMS** e **Geo**

## Estrutura dos Módulos

| Arquivo | Descrição |
|---------|-----------|
| [`qrcode.sea`](qrcode.sea) | Ponto de entrada principal (`qrcode.encode`, `qrcode.decode`, `qrcode.read`, `qrcode.write`, `to_png`, `to_svg`, `to_terminal`) |
| [`encode.sea`](encode.sea) | Codificador QR Code (empacotamento de bits, blocagem G1/G2, entrelaçamento e seleção automática de versão e máscara) |
| [`decode.sea`](decode.sea) | Decodificador de matriz QR (desmascaramento, desentrelaçamento, correção Reed-Solomon e parser de bitstream) |
| [`detector.sea`](detector.sea) | Binarização Otsu/adaptativa, scanner de Finder Patterns `1:1:3:1:1` e extrator de grade com perspectiva |
| [`reed_solomon.sea`](reed_solomon.sea) | Aritmética de corpo finito $\text{GF}(2^8)$ e codificador/decodificador Reed-Solomon |
| [`matrix.sea`](matrix.sea) | Construção da matriz 2D, padrões funcionais (Finders, Alignment, Timing), 8 máscaras e cálculo de penalidade |
| [`tables.sea`](tables.sea) | Tabelas ISO/IEC 18004 para versões 1..40, capacidades, coordenadas de alinhamento e códigos BCH |
| [`image.sea`](image.sea) | Leitura, escrita e renderização em PNG, SVG, BMP, PGM/PPM/PBM, ASCII e Terminal Unicode |
| [`payload.sea`](payload.sea) | Construtores e parsers de payloads PIX (BR Code + CRC16), Wi-Fi, vCard 3.0, TOTP 2FA, Mailto, SMS e Geo |
| [`qrcode_test.sea`](qrcode_test.sea) | Suíte de testes automatizados (`sea test`) |

## Uso Rápido

```sea
local qrcode = require("./qrcode")

-- 1. Gerar um QR Code, exibir no terminal e salvar em PNG / SVG
local qr = qrcode.encode("https://github.com/dopsilva/sea", {ec_level = "M"})?
print(qr:to_terminal())
qr:save("qrcode.png", {scale = 8, border = 4})?
qr:save("qrcode.svg", {scale = 8, border = 4})?

-- 2. Ler um QR Code diretamente de um arquivo PNG, SVG, BMP ou PGM
local texto = qrcode.read("examples/sea_url.png")?
print("Lido:", texto)

-- 3. Decodificação detalhada (versão, máscara, erros corrigidos e payload estruturado)
local res = qrcode.decode("examples/pix.png")?
printf("Texto: %s (V%d-%s, máscara %d, erros corrigidos: %d)\n",
    res.text, res.version, res.ec_level, res.mask, res.errors_corrected)
if res.payload.type == "pix" then
    printf("PIX -> Chave: %s | Nome: %s | Valor: R$ %.2f | CRC válido: %s\n",
        res.payload.key, res.payload.name, res.payload.amount, tostring(res.payload.valid_crc))
end

-- 4. Gerar QR Code PIX (BR Code EMV + CRC16-CCITT)
local brcode = qrcode.payload.pix({
    key = "daniel@sea.dev",
    name = "DANIEL SILVA",
    city = "SAO PAULO",
    amount = 149.90,
    txid = "PEDIDO2026",
})?
qrcode.write(brcode, "meu_pix.png", {ec_level = "M", scale = 6})?
```

## Executando os Testes e Exemplos

Requer a VM da linguagem **[Sea](https://github.com/dopsilva/sea)** (<https://github.com/dopsilva/sea>) compilada (com `libs/compress`):

```bash
# Rodar a suíte de testes
sea test -v

# Executar a demonstração completa
sea examples/demo.sea

# Usar o utilitário CLI para gerar ou ler QR Codes
sea examples/qr_cli.sea write "Olá Sea VM!" qr_ola.png Q
sea examples/qr_cli.sea read qr_ola.png
```
