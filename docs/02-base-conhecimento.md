## Base de Conhecimento

### Dados Utilizados

| Arquivo | Formato | Para que serve no FII Edu? |
| :--- | :--- | :--- |
| `glossario_indicadores.json` | JSON | Fornecer definições exatas de termos técnicos (P/VP, Dividend Yield, Vacância, Cap Rate). |
| `lista_fiis_simulados.csv` | CSV | Conter dados genéricos/simulados de FIIs de exemplo (Tijolo, Papel, Logística) apenas para exercícios de análise didática. |
| `guias_didaticos.md` | Markdown | Textos de apoio com boas práticas de diversificação e estratégias de investimento de longo prazo. |

> **Dica:** Quer um dataset mais robusto? Você pode utilizar datasets públicos do Hugging Face relacionados a finanças, desde que sejam adequados ao contexto do desafio.

### Adaptações nos Dados

* **Você modificou ou expandiu os dados mockados? Descreva aqui.**
  
  Sim. Foi removido completamente os dados originais de transações bancárias e contas correntes do exemplo inicial e criamos uma base de dados totalmente nova e focada no ecossistema de Fundos Imobiliários.

---

### Estratégia de Integração

* **Como os dados são carregados?**
  Os arquivos JSON e CSV contendo o glossário e os dados simulados dos fundos são carregados localmente no início da execução do script do Streamlit utilizando a biblioteca `pandas`. Eles ficam armazenados na memória da sessão (`st.session_state`) e em cache (`st.cache_data`) para consultas rápidas e eficientes.

* **Como os dados são usados no prompt?**
  O agente utiliza uma estratégia de consulta dinâmica (RAG). Quando o usuário digita uma pergunta, o Python faz uma varredura para identificar se algum código de FII da base foi mencionado. Se encontrado, os indicadores exatos daquele fundo são extraídos e injetados diretamente no contexto do `system_prompt` do Ollama. Isso garante que o modelo local responda com dados precisos e sem alucinações.

#### Implementação do Fluxo de Integração (Python)

```python
import streamlit as st
import pandas as pd
import ollama

# Carrega a base de dados simulada de FIIs
@st.cache_data
def carregar_dados_fiis():
    return pd.read_csv("lista_fiis_simulados.csv")

df_fiis = carregar_dados_fiis()

# Função que busca e monta o contexto do fundo consultado
def buscar_contexto_fii(codigo_digitado):
    codigo_digitado = codigo_digitado.upper().strip()
    fii_filtrado = df_fiis[df_fiis['codigo'] == codigo_digitado]
    
    if not fii_filtrado.empty:
        dados = fii_filtrado.iloc[0]
        return (f"Dados reais do fundo {dados['codigo']}: Tipo: {dados['tipo']}, "
                f"Preço: R$ {dados['preco_cota']:.2f}, P/VP: {dados['p_vp']:.2f}, "
                f"Dividend Yield: {dados['dividend_yield']}% a.m.")
    return ""

# Captura a pergunta do usuário no Streamlit
if prompt := st.chat_input("Pergunte sobre um FII (ex: O FTCO11 está barato?)"):
    contexto_fii = ""
    for codigo in df_fiis['codigo']:
        if codigo in prompt.upper():
            contexto_fii = buscar_contexto_fii(codigo)
            break

    # Injeção dinâmica de contexto e diretrizes de segurança no System Prompt
    system_instruction = (
        "Você é o FIIEdu, um assistente puramente educativo sobre Fundos Imobiliários.\n"
        "Regra crítica: Você NUNCA faz recomendações de compra ou venda de ativos.\n"
    )
    if contexto_fii:
        system_instruction += f"\nUse estritamente estes dados para responder: {contexto_fii}"

    # Chamada ao modelo local Ollama
    response = ollama.chat(model='llama3', messages=[
        {"role": "system", "content": system_instruction},
        {"role": "user", "content": prompt}
    ])
    
    st.write(response['message']['content'])

**Fontes de referência utilizadas para a base de conhecimento:**
* Para Glossário FII's, consultar: [Clube FII Glossário](https://www.clubefii.com.br/)
* Para consultar lista de FII's: [Clube FII Lista FII's](https://www.clubefii.com.br/)
* Guia Oficial de FII's: [Guia de Fundos de Investimento Imobiliários (Gov.br/CVM)](https://www.gov.br/investidor/pt-br/educacional/publicacoes-educacionais/guias/guia-de-fundos-de-investimento-imobiliarios.pdf)

---

### Exemplo de Contexto Montado

Mostre um exemplo de como os dados são formatados para o agente.

```text
[CONTEXTO DA BASE DE CONHECIMENTO]
Indicador: P/VP (Preço sobre Valor Patrimonial)
Definição: Indica se o fundo está caro ou barato. P/VP = 1 significa preço justo. Acima de 1 está caro, abaixo de 1 está com desconto.

Exemplo de FII Simulado na Base:
- Código: FTCO11 (Fundo de Tijolo - Comercial)
- Cota atual: R$ 90,00
- Valor Patrimonial por cota: R$ 100,00
- P/VP Calculado: 0,90 (Desconto de 10%)
- Último Dividend Yield: 0,8% a.m.

[PERGUNTA DO USUÁRIO]
"O fundo FTCO11 está barato? O que significa o P/VP dele?"
