# go2apply MPD

**MPD — Manejo de Plantas Daninhas.** Ferramenta web para estruturar o histórico de manejo
de plantas daninhas de forma visual e cronológica, por talhão. Desenvolvida para os clientes
do sistema de consultoria da Equalizagro.

Arquivo único, sem build, sem dependências: `index.html` abre direto no navegador.

---

## Como abrir

Clique duas vezes em `index.html`, ou sirva a pasta:

```bash
python3 -m http.server 8000     # depois acesse http://localhost:8000
```

Códigos de acesso do protótipo: `SRQ-2201` ou `DEMO`.

### Publicar no GitHub Pages

`Settings` → `Pages` → Source: `Deploy from a branch` → Branch: `main` / `root` → `Save`.
Em poucos minutos o app fica em `https://<usuario>.github.io/<repositorio>/`.

Leia antes a seção **Limitações** — o portão de acesso não é autenticação.

---

## O que a ferramenta faz

**Perfil histórico.** Cada faixa é um ciclo agrícola; o mês em que o ciclo começa é
configurável (abril, maio, junho, julho ou janeiro) e trocá-lo reagrupa os manejos.
A cor da faixa mostra o uso do solo — soja, trigo, aveia, milho, cobertura, pousio — e cada
manejo aparece como um marcador na data em que aconteceu. A escala é fixa em pixels por mês,
com rolagem horizontal: a coluna dos ciclos fica travada à esquerda e todas as faixas correm
juntas, então a mesma época fica alinhada de um ano para o outro.

**Fluxograma do ciclo.** A mesma informação lida como sequência de decisões, com o intervalo
em dias entre cada etapa.

**Base de consulta.** Pressão por espécie ao longo dos ciclos, rotação de mecanismos de ação
e a lista de ingredientes ativos do histórico.

**Cadastro** (só para quem tem papel de consultor). Propriedades, talhões e uso do solo,
gravando no Firestore. O cliente registra manejo, mas não altera cadastro.

### Alerta de rotação (grupos HRAC)

Calculado por talhão, a partir da sequência de ciclos em que cada grupo aparece:

| Nível | Critério | Leitura |
|---|---|---|
| **Crítico** | 4 ciclos seguidos ou mais, sequência ainda ativa | seleção de resistência em curso |
| **Atenção** | exatamente 3 ciclos seguidos, ainda ativa | sem folga para um quarto ano |
| **Passado** | já chegou a 3 seguidos, mas foi interrompida | pressão acumulada permanece no registro |

---

## Formato dos dados

Um talhão é um objeto JSON. Para ver um histórico completo no formato real, abra o app e use
**Dados → Copiar** — ele exporta o talhão carregado exatamente como descrito abaixo. O mesmo
botão aceita colar um JSON e substituir o histórico em tela.

```jsonc
{
  "id": "T1",
  "nome": "T1 — Coxilha",
  "ha": 82,
  "solo": "Latossolo vermelho, argiloso",

  // uso do solo: [cultura, início, fim]
  // culturas: soja | trigo | aveia | milho | cobertura | pousio
  "uso": [
    ["aveia", "2021-04-20", "2021-09-25"],
    ["soja",  "2021-11-05", "2022-04-02"]
  ],

  // manejos, em ordem qualquer — o app agrupa por ciclo pela data
  // tipo: des (dessecação) | pre | pos | seq | mec | cul
  "ops": [
    {
      "data": "2021-09-28",
      "tipo": "des",
      "alvos": ["buva", "azevém"],
      "prods": [
        { "nome": "Glifosato 480 SL", "ia": "glifosato", "dose": "3,0 L/ha", "hrac": 9 },
        { "nome": "DMA 806 BR", "ia": "2,4-D", "dose": "1,0 L/ha", "hrac": 4 }
      ],
      "efic": 88,
      "obs": "Aveia bem fechada, pouca daninha em pé."
    }
  ],

  // pressão observada em campo, por ciclo: 1 baixa, 2 média, 3 alta
  "pressao": {
    "2021/22": { "buva": 2, "capim-amargoso": 1, "azevem": 1 }
  }
}
```

Notas de campo:

- `hrac` aceita número (`9`) ou combinação em texto (`"5+15"`) para formulações prontas.
- `efic` é a eficácia observada em campo, em %. Use `0` para pré-emergentes, que são avaliados
  depois. Valores abaixo de 70 aparecem destacados no fluxograma.
- Segmentos de `uso` que atravessam a virada do ciclo são recortados na exibição e marcados
  com borda pontilhada no lado cortado.

---

## Os dois modos

O app roda em dois modos, decididos pelo arquivo `firebase-config.js`:

```js
window.FIREBASE_CONFIG = { apiKey: "COLE-AQUI", ... };
```

**Demonstração** (`apiKey` ainda em `COLE-AQUI`, ou arquivo ausente). Usa os três talhões fictícios embutidos, o portão aceita
`SRQ-2201` ou `DEMO`, e nada é gravado — os manejos registrados vivem na memória da aba
e se perdem ao recarregar. Serve para mostrar a ferramenta sem tocar em dado de cliente.

**Produção.** Cole o `firebaseConfig` do console em `firebase-config.js` e o app passa a exigir e-mail e senha,
ler os talhões do Firestore e gravar cada manejo com autor e data. O procedimento completo
está em [`docs/banco-de-dados.md`](docs/banco-de-dados.md), e as regras de acesso em
[`firestore.rules`](firestore.rules).

O histórico é **append-only**: manejo gravado não se altera nem se apaga, nem pelo
consultor. Corrigir é criar um registro novo e aposentar o antigo. Isso é garantido pela
regra de segurança, não pelo código da página.

---

## Limitações

- **Em modo demonstração não há persistência alguma.** É proposital, mas não confunda:
  sem a configuração preenchida, o botão "Registrar manejo" não guarda nada.
- **O portão do modo demonstração não é autenticação.** O código é comparado no próprio
  JavaScript; qualquer um que abra o código-fonte o encontra.
- **Os dados que acompanham o repositório são fictícios.** A "Fazenda São Roque" e os três
  talhões foram construídos para serem agronomicamente plausíveis e demonstrar padrões reais
  de dependência de glifosato e falha de rotação — mas não são de cliente algum.
- **Não coloque histórico real de cliente no repositório.** Com o Firestore configurado os
  dados ficam no banco, e nada de cliente precisa entrar no versionamento. O `.gitignore`
  já ignora `*.local.json` e `rascunho/`.
- **Banco não é backup.** Agende a exportação descrita em `docs/banco-de-dados.md`.
- Nada aqui constitui recomendação agronômica. Toda aplicação exige receituário emitido por
  profissional habilitado.

---

## Marca e licença

O nome **go2apply**, a sigla **MPD** e a logo (embutida em base64 no `index.html`) são marca
registrada.
O repositório ainda não tem licença definida — sem um arquivo `LICENSE`, o padrão legal é
"todos os direitos reservados", o que pode ser exatamente o que você quer para um produto
proprietário. Se preferir abrir o código, adicione a licença escolhida.
