# Pipeline de automação de relatórios (RPA + Bronze/Gold)

Automação ponta a ponta que substitui a extração manual de relatórios de uma plataforma de atendimento (MKTZap) por um pipeline reproduzível em Python, e consolida os resultados em planilhas consumidas por dashboards no Tableau.

> Projeto de portfólio. Nomes de empresa, clientes, tenants, URLs e caminhos internos foram anonimizados (Tenant A/B/C). Nenhum dado real, credencial ou sessão de navegador está neste repositório.

## O problema

Três operações (tenants) geravam, todo dia, quatro relatórios (Analítico, Enquetes, Tickets e Completo) por download manual na interface web. O resultado era copiado para planilhas históricas que alimentavam dashboards. O processo era lento, sujeito a erro humano e dependia de formatos de data e de tipo que o Tableau Prep precisa receber de forma consistente.

## A solução

Um notebook Jupyter dividido em etapas independentes:

1. **Setup (Célula 1):** imports, janela de datas (1º dia do mês até agora), URLs e tenants num único lugar.
2. **Sessão (Célula 2):** Chrome via Selenium com perfil persistente, então o login (e MFA) é feito uma vez só.
3. **Extração (Células 3 a 7):** para cada tenant, troca de empresa, injeção de datas em formulários AngularJS, geração do relatório, espera na fila de relatórios em background e download. Erros ficam isolados por relatório e tenant.
4. **Camada Gold (Célula 9):** leitura tolerante de CSV (separador, encoding, cabeçalho deslocado), decodificação de entidades HTML, regras de negócio de limpeza e reordenação para o schema oficial. Exportações vazias são detectadas e registradas em vez de gerar erro.
5. **Consolidação (Células 11 a 14):** motor único que aplica schema de tipos e mescla o mês vigente na planilha histórica.
6. **Utilitários (Célula 15 e auditoria):** reparo de histórico a partir de uma cópia íntegra e comparação entre versões.

## Decisões técnicas que valem destaque

- **Tipo é diferente de formato de exibição.** O Excel recebe `datetime` e inteiros nativos; `DD/MM/YYYY HH:MM` é apenas a máscara da célula (`number_format`). Gravar datas como texto era a causa de o Tableau Prep não conseguir calcular.
- **Datas sem ambiguidade.** Conversão com formatos explícitos, nunca `dayfirst=True` sobre texto misto. Isso eliminou a inversão dia/mês que corrompia o histórico.
- **Guilhotina do mês.** A cada execução, tudo que pertence ao mês vigente é removido do histórico e substituído pelo extrato novo, o que torna o processo idempotente e elimina duplicidades.
- **Escrita in-place.** A aba é substituída dentro do mesmo arquivo (openpyxl em modo append), preservando o identificador do arquivo no Google Drive e a conexão do Tableau.
- **Herança de layout.** Cabeçalho exato e formatos de célula são lidos do arquivo de origem e reaplicados, mantendo paridade com a base de produção.
- **Leitura sem inferência.** O histórico é lido célula a célula com openpyxl, para que textos como "n/a" não virem nulos e códigos não ganhem ".0".
- **Correção de mojibake sem efeito colateral.** Só desfaz UTF-8 lido como CP1252 quando o decode é válido, preservando textos corretos como "NÃO" e "CLASSIFICAÇÃO".
- **Regras por relatório.** Enquetes usam a data de envio como chave de janela (é por ela que a plataforma filtra); Tickets são deduplicados por ID.

## Como executar

```bash
pip install -r requirements.txt
```

1. Defina `PASTA_SAIDA` com a pasta onde ficam as planilhas consolidadas (padrão: `./saida`).
2. Ajuste os tenants e a URL da empresa na Célula 1 (`https://EMPRESA.mktzap.com.br`).
3. Abra `mktzap_pipeline.ipynb` e execute as células em ordem. No primeiro uso, faça o login manualmente na janela do Chrome.

Os seletores da interface (`data-test`, `ng-model`, `ga-event`) refletem a plataforma no momento do desenvolvimento e podem precisar de ajuste.

## Stack

Python, pandas, openpyxl, Selenium, Jupyter, Chrome DevTools Protocol (controle de download).
