# Projeto `currais` — contexto

Documento de contexto para qualquer sessão ou agente que trabalhe neste projeto. Leia inteiro antes de codar.

## Modelos e agentes

- **Agente principal (planejamento, arquitetura, revisão, integração):** Claude Opus 5.5 (`claude-opus-5-5`).
- **Subagentes (implementação de módulos, testes, buscas, documentação):** Claude Sonnet 5.5 (`claude-sonnet-5-5`). Ao criar um subagente, passar `model: "sonnet"`.
- Subagentes recebem este arquivo como contexto e trabalham em um módulo por vez, sem editar módulos alheios.

## Quem e por quê

Trabalho na SpaceVis (monitoramento de gado em confinamento com brincos LoRaWAN). Uso polígonos de curral em KML/KMZ para geofencing: um alerta dispara quando o animal sai do curral esperado. Hoje os currais são desenhados à mão no Google Earth, o que é lento. O objetivo é uma ferramenta em Python que gere os currais automaticamente a partir de imagem de satélite. Depois o ajuste final é feito no Google Earth Pro.

## Objetivo

Uma ferramenta (CLI + interface web simples) em que:

1. Informo a área: um polígono em KML/KMZ, ou um bbox, ou desenho num mapa Leaflet.
2. Ela obtém a imagem de satélite da área.
3. Ela detecta as linhas de currais (corredores/cochos) e as cercas entre currais.
4. Ela gera um KMZ para ajuste, contendo:
   - os currais gerados;
   - a imagem de referência como GroundOverlay, em duas camadas: normal e com contraste CLAHE (liga/desliga), para ver as cercas no Google Earth Pro.

## Fonte de imagem (plugável)

- Provider de tiles XYZ configurável. Padrão: Esri World Imagery; alternativa: Bing.
- Entrada de imagem local georreferenciada: GeoTIFF, ou PNG/JPG + JSON com `{north, south, east, west}`. Cobre ortomosaico de drone e prints do Apple Maps, que costuma ter imagem mais recente.
- Montar um mosaico em Web Mercator, zoom 18–19 (~0,3 m/px). Guardar a transformação px↔lat/lon.
- Respeitar os termos de uso de cada provider (não fazer scraping de Google Maps).

## Método (validado manualmente num confinamento real)

Confinamento típico: linhas curvas e paralelas de currais. Cada linha tem este padrão, de norte a sul:
corredor claro (com cocho) → currais (terra marrom-escura) → faixa verde/drenagem → próximo corredor.
Currais medem ~20 m de largura por ~35–45 m de profundidade. As cercas são linhas finas, quase perpendiculares à linha de currais.

1. **Orientação**: estimar a direção dominante das linhas (gradiente/Hough) e rotacionar o mosaico para as linhas ficarem mais ou menos horizontais. Desfazer a rotação no final.
2. **Detectar corredores**:
   - Canal L do espaço de cor LAB, blur gaussiano σ≈3.
   - Para cada coluna (passo de 10 px, média de ±4 px), achar picos claros com `scipy.signal.find_peaks` (height≈115, prominence≈15, distance≈15).
   - Rastrear cada corredor coluna a coluna: prever o y pela inclinação suavizada e aceitar o pico mais próximo dentro de ±12 px.
   - Ajustar um polinômio de grau 4 por corredor com rejeição iterativa de outliers (2,5σ).
   - Os limiares devem ser parâmetros; auto-calibrar pelos percentis da imagem.
3. **Bordas de cada linha**:
   - Borda norte = corredor de cima deslocado perpendicularmente +DN.
   - Borda sul = corredor de baixo deslocado −DS.
   - Valores medidos: DN≈3,6 m e DS≈9,7 m. Se houver polígonos existentes da mesma fazenda, calibrar DN/DS com eles (comparar as bordas dos polígonos com os corredores detectados).
   - Última linha (sem corredor abaixo): borda sul = primeira transição sustentada para claro (média móvel de 15 px acima do limiar).
4. **Pontas das linhas**:
   - Cortar cada linha onde termina a textura de curral (máscara de terra escura), normalmente paralela às estradas diagonais laterais.
   - Permitir corrigir as pontas na interface, com 2 cliques por ponta.
5. **Cercas (divisões entre currais)**:
   - "Retificar" cada linha: reamostrar a borda N e a borda S em 1200 pontos por fração de comprimento de arco, e fazer `cv2.remap` da faixa entre elas. Assim as cercas viram linhas quase verticais.
   - Aplicar CLAHE. Calcular o perfil (mediana de cada coluna entre 12% e 88% da profundidade) e passar um filtro passa-alta.
   - Autocorrelação para achar o período (~20 m). Depois picking de picos/bordas com esse prior (tolerância ±25% do período).
   - Salvar as imagens retificadas com as divisões marcadas, para revisão.
6. **Polígonos**:
   - Cada curral = N(t_i), N(meio), N(t_i+1), S(t_i+1), S(meio), S(t_i).
   - Currais vizinhos compartilham exatamente os mesmos vértices na divisa.
   - Converter para lon/lat com 7 casas decimais.
7. **Saída KMZ**:
   - Nome "CURRAL"; description "linha X, curral Y".
   - Estilo com contorno verde (`ff00ff00`), largura 4, sem preenchimento.
   - Pasta por linha.
   - Se for passado um KMZ existente, mesclar: manter os polígonos originais intactos e colocar os novos numa subpasta.
   - GroundOverlays em `files/` dentro do KMZ.
8. **QA**:
   - PNG de conferência com todos os polígonos sobre a imagem.
   - Relatório: nº de linhas, nº de currais por linha, largura média e desvio.
   - Sinalizar currais com largura fora de ±30% da mediana da linha.

## Entregáveis

- Pacote `currais/` com módulos: `imagery`, `detect_alleys`, `rows`, `fences`, `export_kml`, `qa`.
- CLI: `python -m currais gerar --area area.kml [--imagem img.tif | --tiles esri] [--referencia existentes.kmz] --saida saida.kmz`
- Interface web (Streamlit ou FastAPI + Leaflet): desenhar a área, ver a detecção, arrastar pontas e divisões, exportar o KMZ.
- README em português e testes com uma imagem de exemplo.
- Código comentado. Parâmetros num `config.yaml` (tamanhos em metros, convertidos para px pela resolução).

## Combinado de trabalho

Começar propondo a arquitetura e perguntar o que faltar antes de codar.

## Material já existente

- **Editor manual de baias** (Leaflet, desenho por cliques, importa KML/KMZ, exporta GeoJSON): https://leandrogbm.github.io/baias-confinamento/ — repo `Leandrogbm/baias-confinamento`, fonte local `C:\Users\LENOVO\Downloads\baias_editor.html`. Serve de fallback para corrigir polígonos e traz a lógica de encaixe de imagem por 2 cantos.
- **Print do Apple Maps** de um confinamento (satélite), já cortado: `apple_fazenda.jpg` no repo acima. Centro do link de origem: `-21.309839, -49.322238`, span `0.006558 × 0.012180` (lat × lng). Os limites do recorte foram estimados, não medidos.
- **Como obter imagem do Apple Maps sem print manual:** link `https://maps.apple.com/frame?map=satellite&center=<lat>,<lng>&span=<dlat>,<dlng>` abre o satélite; o `center`/`span` dão os limites para o GroundOverlay. Só funciona bem em Chrome/Edge.
- O usuário não tem conta Apple Developer (sem token MapKit JS); logo, sem API oficial da Apple.
