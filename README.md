# efca-front

Interface web do questionário EFCA (Escala de Fenótipo de Comportamento Alimentar). Conduz o preenchimento dos 16 itens, consulta a [efca-api](https://github.com/Zero-Kng/efca-api) para calcular a pontuação e apresenta o perfil em gráfico radar, com exportação do resultado em imagem.

Aplicação de arquivo único: todo o HTML, CSS e JavaScript vivem em `index.html`. Não há build, bundler, framework nem `node_modules`. Basta servir o arquivo.

## Fluxo

1. **Abertura.** Apresentação do instrumento e da sua origem.
2. **Consentimento.** Explica o que é coletado, para quê, onde fica, por quanto tempo e quem recebe o quê. O avanço exige duas confirmações explícitas: a pessoa avaliada tem 18 anos ou mais, e há consentimento para o uso dos dados.
3. **Identificação.** Nome e idade do respondente. Idade abaixo de 18 anos bloqueia o fluxo.
4. **Dados complementares.** Gênero, altura e peso, todos opcionais, com aceitação de vírgula decimal no formato brasileiro.
5. **Questionário.** Os 16 itens, um por tela, em escala Likert de 1 a 5, com barra de progresso e navegação para trás sem perder as respostas já marcadas.
6. **Resultado.** Gráfico radar com os cinco domínios, valores por domínio e botão de exportação em PNG.

As perguntas não estão escritas no front-end. Elas vêm de `GET /api/questions`, de modo que alterar o instrumento na API atualiza a interface sem tocar neste repositório.

## Dependência da API

Esta interface não funciona sozinha. Ela precisa de uma instância da `efca-api` acessível e com esta origem liberada no CORS.

A URL da API fica em uma constante no topo do bloco de script:

```javascript
const API_BASE_URL = "https://efca-api.onrender.com";
```

A instância pública roda no plano gratuito do Render, que hiberna após um período sem tráfego. A primeira requisição depois da hibernação pode demorar cerca de um minuto. A tela de carregamento cobre esse tempo e oferece nova tentativa em caso de falha.

## Como rodar localmente

O arquivo precisa ser servido por HTTP. Abrir com duplo clique em `file://` faz o navegador bloquear a chamada à API por política de origem.

```bash
python -m http.server 5500
```

Depois acesse `http://localhost:5500`.

A porta 5500 não é arbitrária: é uma das origens que a `efca-api` libera por padrão em desenvolvimento. Para usar outra porta, ajuste a variável `EFCA_ALLOWED_ORIGINS` na API.

Para apontar para uma API local, altere `API_BASE_URL` para `http://localhost:8080`.

## Tratamento de dados e LGPD

Peso, altura e respostas sobre comportamento alimentar são dados referentes à saúde, que a LGPD (art. 5º, II) classifica como dados pessoais sensíveis. O tratamento se apoia no consentimento do titular (art. 11, I), coletado de forma específica e destacada antes de qualquer campo ser exibido.

- **Necessidade.** Só nome e idade são obrigatórios. Gênero, altura e peso não entram no cálculo e por isso são opcionais.
- **Público.** A EFCA foi validada em adultos (Anger, Formoso e Katz, 2022). O fluxo exige confirmação de maioridade e recusa idade abaixo de 18 anos, o que também evita o regime de dado de criança e adolescente do art. 14.
- **Revogação.** Como nada é persistido, fechar a página ou clicar em Refazer descarta tudo e zera o consentimento.


Nome, idade, gênero, altura e peso são coletados apenas para compor o cabeçalho do relatório. **Nenhum desses campos é enviado para a API.** O corpo do `POST /api/responses` contém somente o mapa de respostas:

```json
{ "answers": { "q1": 4, "q2": 2 } }
```

Os dados de identificação existem apenas em variáveis na memória da aba. Não há `localStorage`, `sessionStorage`, cookie ou envio a terceiros. Fechar ou recarregar a página descarta tudo, e o botão de refazer limpa explicitamente os campos.

A exportação em PNG é gerada no próprio navegador e baixada direto para o disco. A imagem nunca passa por servidor.

## Segurança

- A biblioteca html2canvas é carregada do cdnjs com hash SRI (`integrity`) e `crossorigin`, de modo que o navegador recusa o script se o conteúdo servido pela CDN for alterado.

## Acessibilidade

- Cada valor interpolado no HTML passa por `escapeHTML`, incluindo o texto das perguntas vindo da API, o que impede injeção de marcação através da resposta do servidor.
- Os grupos de escala Likert e de gênero usam `role="radiogroup"` com `aria-label`, e o gráfico usa `role="img"` com descrição textual.
- O foco é movido para o primeiro campo a cada etapa, permitindo preencher o questionário inteiro pelo teclado.
- As cores dos cinco domínios seguem a paleta Okabe-Ito, projetada para permanecer distinguível nos tipos mais comuns de daltonismo. O gráfico também rotula cada eixo, de modo que a leitura não depende só da cor.
- Mensagens de erro são exibidas abaixo do campo correspondente, com texto explicando o formato esperado.

## Decisões de projeto

**Arquivo único, sem dependências de build.** O projeto é pequeno e tem um único consumidor. Um bundler adicionaria etapa de compilação, arquivo de configuração e superfície de manutenção sem resolver nenhum problema existente. O custo disso é que `index.html` já passa de 600 linhas, e esse é o limite prático da abordagem.

**Perguntas servidas pela API.** Duplicar o texto dos itens aqui criaria duas fontes de verdade que sairiam de sincronia na primeira alteração do instrumento.

**Gráfico radar em SVG escrito à mão.** Uma biblioteca de gráficos custaria mais bytes que o projeto inteiro para desenhar um pentágono.

**Identificação nunca sai do navegador.** A separação entre o que a interface coleta e o que a API recebe é deliberada. A API é stateless e não persiste nada, então manter dado pessoal fora dela elimina a categoria inteira de risco de vazamento no servidor.

## Limitações conhecidas

- Não há persistência: sair da página no meio do questionário perde o progresso.
- A interface está fixada em tema claro via `color-scheme: light` e não acompanha a preferência do sistema.
- Não há testes automatizados no repositório.
- O consentimento não gera registro, porque não existe servidor que guarde dado de identificação. Se o projeto passar a persistir qualquer coisa, será preciso registrar quando e com qual versão do texto o consentimento foi dado.

## Aviso

Instrumento de autoavaliação com finalidade acadêmica. O resultado é descritivo, não constitui diagnóstico e não substitui avaliação por profissional de saúde qualificado.
