---
name: estudio-capas-tipografia-visual
description: >
  Use esta skill quando precisar gerar capas de livros completas com cálculo
  preciso de lombada, layout aberto (4ª capa + lombada + capa frontal),
  margens de segurança, sangria (bleed) e cruzes de corte. Ativa quando
  o usuário solicita criação de capa, cálculo de lombada, montagem de
  capa aberta para impressão, exportação PDF/X com CMYK, ou sobreposição
  profissional de tipografia sobre arte de fundo com hierarquia visual.
  Entrega arquivos prontos para impressão gráfica e capas digitais.
---

# Skill: Estúdio de Capas e Tipografia Visual

## Propósito

Motor avançado de design gráfico automatizado focado na engenharia editorial
da embalagem do livro. Integra geração visual com cálculo matemático rigoroso
de layout para impressão comercial, entregando capas profissionais em
múltiplos formatos industriais.

## O que esta skill NÃO faz

- Não gera ilustrações do zero — recebe arte pronta ou gerada por outra skill
- Não faz retoque fotográfico avançado (use a skill de Direção de Arte)
- Não gerenciacatálogos de capas anteriores
- Não imprime — entrega apenas arquivos finais para impressão

## Pré-requisitos

### Dependências Python

```bash
pip install Pillow reportlab numpy
```

### Bibliotecas do Sistema (para fontes)

```bash
# Ubuntu/Debian
sudo apt-get install fonts-liberation fonts-dejavu-core

# macOS
brew install fontforge
```

## Referência do Papel — Gramatura e Espessura

| Papel | Gramatura (g/m²) | Espessura por folha (mm) | Espessura por página (mm) |
|-------|------------------|--------------------------|---------------------------|
| Couché brilho | 115 | 0,096 | 0,048 |
| Couché brilho | 150 | 0,128 | 0,064 |
| Couché brilho | 200 | 0,179 | 0,0895 |
| Couché fosco | 115 | 0,097 | 0,0485 |
| Couché fosco | 150 | 0,130 | 0,065 |
| Couché fosco | 200 | 0,180 | 0,090 |
| Offset | 75 | 0,090 | 0,045 |
| Offset | 90 | 0,108 | 0,054 |
| Rosa | 52 | 0,070 | 0,035 |
| Rosa | 60 | 0,080 | 0,040 |
| Pólen soft | 90 | 0,120 | 0,060 |

> **Nota:** Valores aproximados. Para impressão profissional, sempre confirme
> com a gráfica a espessura exata do papel utilizado.

## Fluxo Principal

### Passo 1 — Coletar especificações do projeto

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class CoverSpecs:
    """Especificações físicas do livro para cálculo de capa."""

    # Dimensões do corte (em milímetros)
    largura_capa_mm: float       # Ex: 160 (para capa brochura 14x21)
    altura_capa_mm: float        # Ex: 230

    # Papel
    gramatura_papel: float       # Ex: 150 (g/m²)
    tipo_papel: str = "couché brilho"  # couché brilho, couché fosco, offset, rosa, pólen

    # Contagem de páginas
    total_paginas: int = 0       # Ex: 320

    # Sangria e margens (em mm)
    bleed_mm: float = 3.0        # Padrão gráficas brasileiras
    margem_seguranca_mm: float = 5.0  # Distância mínima do corte

    # Resolução de saída
    dpi: int = 300               # Padrão industrial

    # Cores
    perfil_cor: str = "CMYK"     # CMYK para impressão, RGB para digital

    # Overlay tipográfico
    titulo: str = ""
    subtitulo: str = ""
    autor: str = ""
    editora: str = ""
    logo_editora_path: Optional[str] = None
```

### Passo 2 — Calcular espessura da lombada

```python
# Tabela de espessuras por gramatura (mm por página)
ESPESSURA_POR_GRAMATURA = {
    # gramatura: (espessura_por_folha_mm, espessura_por_pagina_mm)
    52:  (0.070, 0.035),
    60:  (0.080, 0.040),
    75:  (0.090, 0.045),
    90:  (0.108, 0.054),   # offset e pólen
    115: (0.096, 0.048),   # couché
    150: (0.128, 0.064),   # couché
    200: (0.179, 0.0895),  # couché
}

def calcular_lombada(specs: CoverSpecs) -> float:
    """
    Calcula a largura exata da lombada em milímetros.

    Fórmula: lombada = (total_paginas / 2) × espessura_por_folha
    + margem para cola (0,5 mm por lado da lombada)

    Retorna: largura da lombada em mm
    """
    gramatura = specs.gramatura_papel

    if gramatura in ESPESSURA_POR_GRAMATURA:
        espessura_folha, espessura_pagina = ESPESSURA_POR_GRAMATURA[gramatura]
    else:
        # Interpolação linear para gramaturas não listadas
        gramaturas = sorted(ESPESSURA_POR_GRAMATURA.keys())
        if gramatura < gramaturas[0] or gramatura > gramaturas[-1]:
            raise ValueError(
                f"Gramatura {gramatura} g/m² fora da faixa de referência "
                f"({gramaturas[0]}-{gramaturas[-1]} g/m²). "
                f"Forneça a espessura do papel manualmente."
            )

        # Encontra as gramaturas vizinhas
        for i in range(len(gramaturas) - 1):
            if gramaturas[i] <= gramatura <= gramaturas[i + 1]:
                g1, g2 = gramaturas[i], gramaturas[i + 1]
                e1_folha, e1_pag = ESPESSURA_POR_GRAMATURA[g1]
                e2_folha, e2_pag = ESPESSURA_POR_GRAMATURA[g2]

                # Interpolação
                fator = (gramatura - g1) / (g2 - g1)
                espessura_folha = e1_folha + fator * (e2_folha - e1_folha)
                espessura_pagina = e1_pag + fator * (e2_pag - e1_pag)
                break

    # Número de folhas = total de páginas / 2
    num_folhas = specs.total_paginas / 2

    # Largura da lombada = folhas × espessura + cola
    # Adiciona 1mm para compensação de cola (0,5mm por lado)
    lombada_mm = (num_folhas * espessura_folha) + 1.0

    return round(lombada_mm, 2)


def calcular_lombada_com_margem(
    specs: CoverSpecs,
    margem_lombada_mm: float = 1.0
) -> dict:
    """
    Calcula lombada com margem para costura/cola.

    Retorna dict com valores detalhados.
    """
    lombada_base = calcular_lombada(specs)

    return {
        "lombada_base_mm": lombada_base,
        "margem_cola_mm": margem_lombada_mm,
        "lombada_total_mm": lombada_base + margem_lombada_mm,
        "gramatura": specs.gramatura_papel,
        "total_paginas": specs.total_paginas,
        "papel": specs.tipo_papel,
    }
```

### Passo 3 — Montar geometria da capa aberta

```python
@dataclass
class CoverGeometry:
    """Geometria completa da capa aberta para impressão."""

    # Dimensões totais do arquivo (em pixels)
    largura_total_px: int
    altura_total_px: int

    # Dimensões totais em mm
    largura_total_mm: float
    altura_total_mm: float

    # Posições das zonas (em pixels, relativas à origem do arquivo)
    capa_frontal_x: int
    capa_frontal_largura: int

    lombada_x: int
    lombada_largura: int

    quarta_capa_x: int
    quarta_capa_largura: int

    # Sangria
    bleed_px: int

    # Margem de segurança
    seguranca_px: int


def montar_geometria_capa(specs: CoverSpecs) -> CoverGeometry:
    """
    Monta a geometria completa da capa aberta.

    Layout (visto de frente, com a capa frontal à direita):

    [  BLEED  ][  QUARTA CAPA  ][ LOMBADA ][  CAPA FRONTAL  ][  BLEED  ]
    [  BLEED  ][  SEGURANÇA    ][ SEGUR.   ][  SEGURANÇA     ][  BLEED  ]

    Retorna CoverGeometry com todas as posições e dimensões em pixels.
    """
    dpi = specs.dpi
    fator_mm_para_px = dpi / 25.4  # Conversão mm → pixels

    # Dimensões base
    largura_capa_px = round(specs.largura_capa_mm * fator_mm_para_px)
    altura_capa_px = round(specs.altura_capa_mm * fator_mm_para_px)
    bleed_px = round(specs.bleed_mm * fator_mm_para_px)
    seguranca_px = round(specs.margem_seguranca_mm * fator_mm_para_px)

    # Largura da lombada
    lombada_info = calcular_lombada_com_margem(specs)
    lombada_mm = lombada_info["lombada_total_mm"]
    lombada_px = round(lombada_mm * fator_mm_para_px)

    # Largura total = bleed esquerdo + quarta capa + lombada + capa frontal + bleed direito
    largura_total_px = bleed_px + largura_capa_px + lombada_px + largura_capa_px + bleed_px

    # Altura total = bleed superior + altura capa + bleed inferior
    altura_total_px = bleed_px + altura_capa_px + bleed_px

    # Posições X (a partir da esquerda do arquivo)
    quarta_capa_x = bleed_px
    lombada_x = bleed_px + largura_capa_px
    capa_frontal_x = bleed_px + largura_capa_px + lombada_px

    # Dimensões em mm
    largura_total_mm = specs.largura_capa_mm * 2 + lombada_mm + (specs.bleed_mm * 2)
    altura_total_mm = specs.altura_capa_mm + (specs.bleed_mm * 2)

    return CoverGeometry(
        largura_total_px=largura_total_px,
        altura_total_px=altura_total_px,
        largura_total_mm=largura_total_mm,
        altura_total_mm=altura_total_mm,
        capa_frontal_x=capa_frontal_x,
        capa_frontal_largura=largura_capa_px,
        lombada_x=lombada_x,
        lombada_largura=lombada_px,
        quarta_capa_x=quarta_capa_x,
        quarta_capa_largura=largura_capa_px,
        bleed_px=bleed_px,
        seguranca_px=seguranca_px,
    )
```

### Passo 4 — Gerar imagem base da capa aberta

```python
from PIL import Image, ImageDraw, ImageFont
import os

def criar_base_capa_aberta(geometry: CoverGeometry, cor_fundo: str = "#FFFFFF") -> Image.Image:
    """
    Cria a imagem base da capa aberta com o fundo指定ido.

    Retorna imagem PIL no modo RGB pronta para receber sobreposições.
    """
    img = Image.new(
        "RGB",
        (geometry.largura_total_px, geometry.altura_total_px),
        color=cor_fundo
    )
    return img


def desenhar_guias_estrutura(img: Image.Image, geometry: CoverGeometry) -> Image.Image:
    """
    Desenha guias visuais sobre a imagem:
    - Linhas vermelhas para limites de sangria
    - Linhas azuis para margem de segurança
    - Linhas verdes para limites da lombada

    Retorna cópia da imagem com as guias.
    """
    img_guia = img.copy()
    draw = ImageDraw.Draw(img_guia)

    bleed = geometry.bleed_px
    seg = geometry.seguranca_px
    largura = geometry.largura_total_px
    altura = geometry.altura_total_px

    # Limites de sangria (vermelho)
    draw.rectangle([bleed, bleed, largura - bleed, altura - bleed], outline="red", width=2)

    # Margem de segurança (azul)
    seg_inicial = bleed + seg
    draw.rectangle(
        [seg_inicial, seg_inicial, largura - seg_inicial, altura - seg_inicial],
        outline="blue", width=2
    )

    # Limites da lombada (verde)
    draw.line([(geometry.lombada_x, 0), (geometry.lombada_x, altura)], fill="green", width=2)
    draw.line([(geometry.lombada_x + geometry.lombada_largura, 0),
               (geometry.lombada_x + geometry.lombada_largura, altura)], fill="green", width=2)

    return img_guia


def desenhar_cruzes_corte(img: Image.Image, geometry: CoverGeometry) -> Image.Image:
    """
    Desenha cruzes de marca de corte (crop marks) nos cantos da área de corte.

    Conforme padrão gráfico: linhas de 3mm de extensão, 0,3mm de espessura,
    posicionadas 3mm fora da área de corte.
    """
    img_corte = img.copy()
    draw = ImageDraw.Draw(img_corte)

    bleed = geometry.bleed_px
    margem_cruz_px = round(3 * (geometry.largura_total_px / geometry.largura_total_mm))  # 3mm em pixels
    comprimento_cruz_px = round(5 * (geometry.largura_total_px / geometry.largura_total_px))  # 5mm

    # Cantos da área de corte (área visível = sem bleed)
    cantos = [
        (bleed, bleed),                                    # Superior esquerdo
        (geometry.largura_total_px - bleed, bleed),        # Superior direito
        (bleed, geometry.altura_total_px - bleed),         # Inferior esquerdo
        (geometry.largura_total_px - bleed,
         geometry.altura_total_px - bleed),                # Inferior direito
    ]

    espessura = max(1, round(0.3 * (300 / 25.4)))  # 0,3mm em pixels a 300dpi

    for canto_x, canto_y in cantos:
        # Horizontal
        if canto_x == bleed:  # Lado esquerdo
            draw.line(
                [(canto_x - margem_cruz_px - comprimento_cruz_px, canto_y),
                 (canto_x - margem_cruz_px, canto_y)],
                fill="black", width=espessura
            )
        else:  # Lado direito
            draw.line(
                [(canto_x + margem_cruz_px, canto_y),
                 (canto_x + margem_cruz_px + comprimento_cruz_px, canto_y)],
                fill="black", width=espessura
            )

        # Vertical
        if canto_y == bleed:  # Superior
            draw.line(
                [(canto_x, canto_y - margem_cruz_px - comprimento_cruz_px),
                 (canto_x, canto_y - margem_cruz_px)],
                fill="black", width=espessura
            )
        else:  # Inferior
            draw.line(
                [(canto_x, canto_y + margem_cruz_px),
                 (canto_x, canto_y + margem_cruz_px + comprimento_cruz_px)],
                fill="black", width=espessura
            )

    return img_corte
```

### Passo 5 — Motor de renderização tipográfica

```python
class MotorTipografia:
    """
    Motor de renderização de fontes para sobreposição profissional
    de textos sobre arte de fundo.
    """

    # Hierarquia visual padrão para capas de livros
    HIERARQUIA_PADRAO = {
        "titulo": {"tamanho_mm": 18, "peso": "bold", "alinhamento": "center"},
        "subtitulo": {"tamanho_mm": 10, "peso": "regular", "alinhamento": "center"},
        "autor": {"tamanho_mm": 12, "peso": "bold", "alinhamento": "center"},
        "editora": {"tamanho_mm": 8, "peso": "regular", "alinhamento": "center"},
    }

    def __init__(self, dpi: int = 300):
        self.dpi = dpi
        self.fator_mm_px = dpi / 25.4
        self._fontes_cache = {}

    def _carregar_fonte(self, caminho_fonte: str, tamanho_px: int) -> ImageFont.FreeTypeFont:
        """Carrega fonte TrueType com cache."""
        chave = (caminho_fonte, tamanho_px)
        if chave not in self._fontes_cache:
            try:
                self._fontes_cache[chave] = ImageFont.truetype(caminho_fonte, tamanho_px)
            except (IOError, OSError) as e:
                print(f"⚠️ Fonte não encontrada: {caminho_fonte}. Usando padrão.")
                self._fontes_cache[chave] = ImageFont.load_default()
        return self._fontes_cache[chave]

    def calcular_tamanho_px(self, tamanho_mm: int) -> int:
        """Converte tamanho de mm para pixels."""
        return round(tamanho_mm * self.fator_mm_px)

    def medir_texto(
        self,
        texto: str,
        fonte: ImageFont.FreeTypeFont
    ) -> tuple[int, int]:
        """Mede largura e altura do texto em pixels."""
        bbox = fonte.getbbox(texto)
        return bbox[2] - bbox[0], bbox[3] - bbox[1]

    def calcular_contraste_cor(
        self,
        cor_texto: tuple,
        cor_fundo: tuple
    ) -> float:
        """
        Calcula o nível de contraste entre cor de texto e fundo.
        Usa a fórmula de luminância relativa WCAG 2.0.

        Retorna valor entre 0 (sem contraste) e 21 (contraste máximo).
        """
        def luminancia_relativa(cor: tuple) -> float:
            r, g, b = [c / 255.0 for c in cor[:3]]
            r_lin = r / 12.92 if r <= 0.03928 else ((r + 0.055) / 1.055) ** 2.4
            g_lin = g / 12.92 if g <= 0.03928 else ((g + 0.055) / 1.055) ** 2.4
            b_lin = b / 12.92 if b <= 0.03928 else ((b + 0.055) / 1.055) ** 2.4
            return 0.2126 * r_lin + 0.7152 * g_lin + 0.0722 * b_lin

        l1 = luminancia_relativa(cor_fundo)
        l2 = luminancia_relativa(cor_texto)
        contraste = (max(l1, l2) + 0.05) / (min(l1, l2) + 0.05)
        return round(contraste, 2)

    def sugerir_cor_texto(
        self,
        cor_fundo: tuple,
        contraste_minimo: float = 4.5
    ) -> tuple:
        """
        Sugerir cor de texto com contraste adequado sobre o fundo.

        Tenta preto e branco; se nenhum atingir o contraste mínimo,
        retorna branco com aviso.
        """
        preto = (0, 0, 0)
        branco = (255, 255, 255)

        contraste_preto = self.calcular_contraste_cor(preto, cor_fundo)
        contraste_branco = self.calcular_contraste_cor(branco, cor_fundo)

        if contraste_preto >= contraste_minimo:
            return preto
        elif contraste_branco >= contraste_minimo:
            return branco
        else:
            print(f"⚠️ Nenhuma cor padrão atinge contraste {contraste_minimo}. Usando branco.")
            return branco

    def aplicar_sombra_texto(
        self,
        img: Image.Image,
        posicao: tuple[int, int],
        texto: str,
        fonte: ImageFont.FreeTypeFont,
        cor_sombra: tuple = (0, 0, 0),
        deslocamento: tuple[int, int] = (2, 2),
        opacidade: int = 128
    ) -> Image.Image:
        """
        Aplica texto com sombra projetada para melhorar legibilidade.
        """
        img_sombra = Image.new("RGBA", img.size, (0, 0, 0, 0))
        draw_sombra = ImageDraw.Draw(img_sombra)

        # Desenha sombra
        sombra_x = posicao[0] + deslocamento[0]
        sombra_y = posicao[1] + deslocamento[1]
        draw_sombra.text(
            (sombra_x, sombra_y),
            texto,
            font=fonte,
            fill=cor_sombra + (opacidade,)
        )

        # Combinar com imagem original
        if img.mode != "RGBA":
            img = img.convert("RGBA")

        return Image.alpha_composite(img, img_sombra)

    def renderizar_texto_na_capa(
        self,
        img: Image.Image,
        texto: str,
        tipo_elemento: str,  # titulo, subtitulo, autor, editora
        posicao: tuple[int, int],
        caminho_fonte: str,
        cor_texto: tuple = (0, 0, 0),
        cor_sombra: tuple = (0, 0, 0),
        usar_sombra: bool = False,
        alinhamento: str = "center"
    ) -> Image.Image:
        """
        Renderiza um elemento de texto na capa com hierarquia visual.

        Parâmetros:
            img: Imagem da capa
            texto: Texto a renderizar
            tipo_elemento: Um de [titulo, subtitulo, autor, editora]
            posicao: Tupla (x, y) central do texto
            caminho_fonte: Caminho completo para arquivo .ttf/.otf
            cor_texto: Tupla RGB
            usar_sombra: Se True, aplica sombra projetada
            alinhamento: "center", "left" ou "right"
        """
        hierarquia = self.HIERARQUIA_PADRAO.get(tipo_elemento, self.HIERARQUIA_PADRAO["titulo"])
        tamanho_mm = hierarquia["tamanho_mm"]
        tamanho_px = self.calcular_tamanho_px(tamanho_mm)

        fonte = self._carregar_fonte(caminho_fonte, tamanho_px)
        largura_texto, altura_texto = self.medir_texto(texto, fonte)

        # Calcular posição com alinhamento
        if alinhamento == "center":
            x_texto = posicao[0] - (largura_texto // 2)
        elif alinhamento == "right":
            x_texto = posicao[0] - largura_texto
        else:  # left
            x_texto = posicao[0]

        y_texto = posicao[1] - (altura_texto // 2)

        # Converter para RGBA se necessário
        if img.mode != "RGBA":
            img = img.convert("RGBA")

        # Aplicar sombra se solicitado
        if usar_sombra:
            img = self.aplicar_sombra_texto(
                img, (x_texto, y_texto), texto, fonte,
                cor_sombra=cor_sombra
            )

        # Desenhar texto principal
        draw = ImageDraw.Draw(img)
        draw.text((x_texto, y_texto), texto, font=fonte, fill=cor_texto + (255,))

        return img
```

### Passo 6 — Sobreposição de arte de fundo

```python
def aplicar_arte_fundo(
    img_capa: Image.Image,
    arte_path: str,
    geometry: CoverGeometry,
    zona: str = "capa_frontal",
    modo_ajuste: str = "preencher"
) -> Image.Image:
    """
    Aplica arte de fundo sobre uma zona da capa aberta.

    Parâmetros:
        img_capa: Imagem da capa aberta
        arte_path: Caminho para a imagem de arte
        geometry: Geometria da capa
        zona: "capa_frontal", "quarta_capa" ou "lombada"
        modo_ajuste: "preencher" (cover), "conter" (contain) ou "esticar" (stretch)
    """
    try:
        arte = Image.open(arte_path)
    except (IOError, OSError) as e:
        print(f"⚠️ Erro ao abrir arte: {e}. Retornando capa sem alteração.")
        return img_capa

    # Determinar zona alvo
    if zona == "capa_frontal":
        x_inicio = geometry.capa_frontal_x
        largura_zona = geometry.capa_frontal_largura
    elif zona == "quarta_capa":
        x_inicio = geometry.quarta_capa_x
        largura_zona = geometry.quarta_capa_largura
    elif zona == "lombada":
        x_inicio = geometry.lombada_x
        largura_zona = geometry.lombada_largura
    else:
        raise ValueError(f"Zona inválida: {zona}. Use: capa_frontal, quarta_capa, lombada")

    altura_zona = geometry.altura_total_px

    # Redimensionar arte para a zona
    if modo_ajuste == "preencher":
        # Redimensionar para preencher (crop se necessário)
        ratio_largura = largura_zona / arte.width
        ratio_altura = altura_zona / arte.height
        ratio = max(ratio_largura, ratio_altura)

        nova_largura = round(arte.width * ratio)
        nova_altura = round(arte.height * ratio)
        arte = arte.resize((nova_largura, nova_altura), Image.Resampling.LANCZOS)

        # Centralizar e recortar
        offset_x = (nova_largura - largura_zona) // 2
        offset_y = (nova_altura - altura_zona) // 2
        arte = arte.crop((offset_x, offset_y, offset_x + largura_zona, offset_y + altura_zona))

    elif modo_ajuste == "conter":
        # Redimensionar para caber (com bordas)
        ratio_largura = largura_zona / arte.width
        ratio_altura = altura_zona / arte.height
        ratio = min(ratio_largura, ratio_altura)

        nova_largura = round(arte.width * ratio)
        nova_altura = round(arte.height * ratio)
        arte = arte.resize((nova_largura, nova_altura), Image.Resampling.LANCZOS)

        # Centralizar na zona
        offset_x = (largura_zona - nova_largura) // 2
        offset_y = (altura_zona - nova_altura) // 2

        # Criar camada temporária
        arte_posicionada = Image.new("RGB", (largura_zona, altura_zona), (255, 255, 255))
        arte_posicionada.paste(arte, (offset_x, offset_y))
        arte = arte_posicionada

    elif modo_ajuste == "esticar":
        arte = arte.resize((largura_zona, altura_zona), Image.Resampling.LANCZOS)

    # Colar arte na zona
    img_capa.paste(arte, (x_inicio, 0))

    return img_capa
```

### Passo 7 — Exportação em múltiplos formatos

```python
from reportlab.lib.units import mm
from reportlab.pdfgen import canvas
from reportlab.lib.utils import ImageReader
import io

def exportar_pdf_impressao(
    img: Image.Image,
    geometry: CoverGeometry,
    caminho_saida: str,
    specs: CoverSpecs
) -> str:
    """
    Exporta a capa como PDF/X para impressão com perfil CMYK.

    Gera PDF com dimensões exatas em mm, incluindo sangria.
    """
    try:
        # Converter para CMYK (simulação — para PDF/X real, usar ICC profile)
        if img.mode != "CMYK":
            img_cmyk = img.convert("CMYK")
        else:
            img_cmyk = img.copy()

        # Criar PDF com ReportLab
        largura_mm = geometry.largura_total_mm
        altura_mm = geometry.altura_total_mm

        c = canvas.Canvas(caminho_saida, pagesize=(largura_mm * mm, altura_mm * mm))

        # Salvar imagem temporária em PNG de alta qualidade
        buf = io.BytesIO()
        img_cmyk.save(buf, format="PNG", dpi=(300, 300))
        buf.seek(0)

        # Adicionar imagem ao PDF
        img_reader = ImageReader(buf)
        c.drawImage(
            img_reader,
            0, 0,
            width=largura_mm * mm,
            height=altura_mm * mm
        )

        # Metadados
        c.setTitle("Capa - PDF/X para Impressão")
        c.setAuthor("EscreveAI - Estúdio de Capas")
        c.setSubject(f"Capa aberta - {specs.largura_capa_mm}x{specs.altura_capa_mm}mm")

        c.save()

        print(f"✅ PDF de impressão salvo: {caminho_saida}")
        print(f"   Dimensões: {largura_mm:.1f} x {altura_mm:.1f} mm")
        print(f"   Resolução: {specs.dpi} DPI")
        print(f"   Perfil: CMYK (simulado)")

        return caminho_saida

    except Exception as e:
        print(f"❌ Erro ao exportar PDF de impressão: {e}")
        raise


def exportar_capa_digital(
    img: Image.Image,
    caminho_saida: str,
    specs: CoverSpecs,
    largura_max_px: int = 1600
) -> str:
    """
    Exporta capa digital em RGB para e-commerce e leitores digitais.
    """
    try:
        # Converter para RGB se necessário
        if img.mode != "RGB":
            img_rgb = img.convert("RGB")
        else:
            img_rgb = img.copy()

        # Redimensionar se maior que largura máxima
        if img_rgb.width > largura_max_px:
            ratio = largura_max_px / img_rgb.width
            nova_altura = round(img_rgb.height * ratio)
            img_rgb = img_rgb.resize((largura_max_px, nova_altura), Image.Resampling.LANCZOS)

        # Salvar como PNG de alta qualidade
        img_rgb.save(caminho_saida, "PNG", optimize=True)

        print(f"✅ Capa digital salva: {caminho_saida}")
        print(f"   Dimensões: {img_rgb.width}x{img_rgb.height} px")
        print(f"   Perfil: RGB")

        return caminho_saida

    except Exception as e:
        print(f"❌ Erro ao exportar capa digital: {e}")
        raise
```

### Passo 8 — Fluxo completo de geração

```python
def gerar_capa_completa(
    specs: CoverSpecs,
    arte_fundo_path: str,
    caminho_saida_pasta: str,
    usar_guias: bool = False,
    usar_cruzes_corte: bool = True
) -> dict:
    """
    Fluxo completo: calcula lombada → monta geometria → aplica arte →
    renderiza tipografia → exporta PDF/X e versão digital.

    Retorna dict com caminhos dos arquivos gerados.
    """
    import os

    # Criar pasta de saída
    os.makedirs(caminho_saida_pasta, exist_ok=True)

    # 1. Calcular geometria
    geometry = montar_geometria_capa(specs)
    lombada_info = calcular_lombada_com_margem(specs)

    print(f"📐 Geometria calculada:")
    print(f"   Lombada: {lombada_info['lombada_total_mm']:.2f} mm")
    print(f"   Arquivo total: {geometry.largura_total_mm:.1f} x {geometry.altura_total_mm:.1f} mm")
    print(f"   Arquivo pixels: {geometry.largura_total_px}x{geometry.altura_total_px} px")

    # 2. Criar base
    img = criar_base_capa_aberta(geometry)

    # 3. Aplicar arte de fundo na capa frontal
    img = aplicar_arte_fundo(img, arte_fundo_path, geometry, "capa_frontal")

    # 4. Aplicar arte na quarta capa (se disponível)
    # Arte da quarta capa pode ser a mesma ou diferente

    # 5. Desenhar guias (opcional)
    if usar_guias:
        img = desenhar_guias_estrutura(img, geometry)

    # 6. Desenhar cruzes de corte
    if usar_cruzes_corte:
        img = desenhar_cruzes_corte(img, geometry)

    # 7. Renderizar tipografia
    motor = MotorTipografia(dpi=specs.dpi)

    # Posições tipográficas (centralizadas na capa frontal)
    centro_x = geometry.capa_frontal_x + (geometry.capa_frontal_largura // 2)

    # Título (40% da altura da capa, a partir do topo)
    pos_titulo_y = round(geometry.altura_total_px * 0.35)
    if specs.titulo:
        img = motor.renderizar_texto_na_capa(
            img, specs.titulo, "titulo",
            (centro_x, pos_titulo_y),
            caminho_fonte="/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
            usar_sombra=True
        )

    # Subtítulo
    pos_subtitulo_y = round(geometry.altura_total_px * 0.45)
    if specs.subtitulo:
        img = motor.renderizar_texto_na_capa(
            img, specs.subtitulo, "subtitulo",
            (centro_x, pos_subtitulo_y),
            caminho_fonte="/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
            usar_sombra=True
        )

    # Autor (parte inferior)
    pos_autor_y = round(geometry.altura_total_px * 0.88)
    if specs.autor:
        img = motor.renderizar_texto_na_capa(
            img, specs.autor, "autor",
            (centro_x, pos_autor_y),
            caminho_fonte="/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
            usar_sombra=True
        )

    # Editora (rodapé)
    pos_editora_y = round(geometry.altura_total_px * 0.95)
    if specs.editora:
        img = motor.renderizar_texto_na_capa(
            img, specs.editora, "editora",
            (centro_x, pos_editora_y),
            caminho_fonte="/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
            usar_sombra=False
        )

    # 8. Exportar
    resultado = {}

    # PDF para impressão
    pdf_path = os.path.join(caminho_saida_pasta, "capa_impressao.pdf")
    exportar_pdf_impressao(img, geometry, pdf_path, specs)
    resultado["pdf_impressao"] = pdf_path

    # Versão digital
    digital_path = os.path.join(caminho_saida_pasta, "capa_digital.png")
    exportar_capa_digital(img, digital_path, specs)
    resultado["capa_digital"] = digital_path

    # Versão com guias (para revisão)
    if usar_guias:
        img_com_guias = desenhar_guias_estrutura(img, geometry)
        img_com_guias = desenhar_cruzes_corte(img_com_guias, geometry)
        guias_path = os.path.join(caminho_saida_pasta, "capa_com_guias.png")
        img_com_guias.save(guias_path, "PNG")
        resultado["versao_guias"] = guias_path

    return resultado
```

## Tabela de Referência — Dimensões Comuns

| Formato | Capa (mm) | Altura (mm) | Observação |
|---------|-----------|-------------|------------|
| Brochura 14×21 | 160 | 230 | Mais comum Brasil |
| Brochura 14×22 | 160 | 240 | Narrativa/romance |
| Brochura 16×23 | 180 | 250 | Didáticos |
| Capa dura 16×23 | 185 | 255 | +3mm cada lado |
| ebook | N/A | 1600px | Largura fixa |

## Regras

### O que SEMPRE fazer

- Calcular lombada antes de montar o layout — nunca usar valor fixo
- Incluir sangria (bleed) de 3mm para gráficas brasileiras (confirmar com a gráfica)
- Adicionar margem de segurança de 5mm para textos e elementos importantes
- Incluir cruzes de corte em arquivos para impressão profissional
- Verificar contraste entre tipografia e fundo antes de renderizar
- Exportar em CMYK para impressão e RGB para digital
- Usar resolução mínima de 300 DPI para impressão

### O que NUNCA fazer

- Usar lombada fixa sem calcular com base na gramatura e páginas
- Posicionar texto dentro da área de sangria (fora do corte)
- Exportar para impressão em RGB
- Usar resolução inferior a 300 DPI para impressão
- Ignorar margem de segurança — o texto pode ser cortado
- Sobrepor tipografia sobre arte com baixo contraste sem aplicar sombra/máscara

### Quando algo falhar

- **Arquivo de arte não encontrado:** Informar ao usuário e pedir caminho correto
- **Gramatura não encontrada na tabela:** Solicitar espessura manual do papel ou usar interpolação com aviso
- **Fonte não encontrada:** Usar fonte padrão (DejaVu) com aviso — nunca falhar silenciosamente
- **Contraste insuficiente:** Aplicar automaticamente sombra ou outline — informar ao usuário
- **Erro de exportação:** Informar detalhes do erro e sugerir verificar permissões de escrita

## Nota sobre Perfis ICC

Para PDF/X verdadeiro com perfil CMYK certificado, é necessário instalar perfis ICC
(fornecidos pela gráfica) e usar bibliotecas como `littlecms` ou `Pillow` com ICC.
Esta implementação faz conversão simulada. Para produção profissional, solicite o
perfil ICC da gráfica e integre usando:

```python
# Exemplo com Pillow + perfil ICC
from PIL import ImageCms

def aplicar_perfil_cmyk(img_rgb: Image.Image, perfil_icc_path: str) -> Image.Image:
    """Converte RGB para CMYK usando perfil ICC específico."""
    try:
        perfil_entrada = ImageCms.createProfile("sRGB")
        perfil_saida = ImageCms.getOpenProfile(perfil_icc_path)
        transformacao = ImageCms.buildTransform(
            perfil_entrada, perfil_saida, "RGB", "CMYK"
        )
        return ImageCms.applyTransform(img_rgb, transformacao)
    except Exception as e:
        print(f"⚠️ Erro ao aplicar perfil ICC: {e}. Usando conversão padrão.")
        return img_rgb.convert("CMYK")
```

---

*Skill: Estúdio de Capas e Tipografia Visual — EscreveAI*
