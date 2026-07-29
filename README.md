# aurora-pylibs

O catálogo de bibliotecas Python da [AURORA](https://github.com/nipscernlab/aurora),
a IDE do ecossistema SAPHO do NIPS-CERN.

Este repositório guarda **a lista**, não as bibliotecas. O `catalog.json` descreve
quais bibliotecas o painel da AURORA oferece, para que cada uma serve e qual
versão está fixada. Os arquivos em si continuam vindo da PyPI no momento da
instalação, conferidos contra o `sha256` que este catálogo fixou.

A vantagem de a lista morar aqui é o ritmo: dá para acrescentar ou tirar uma
biblioteca sem lançar uma versão nova da AURORA. A IDE busca este arquivo ao
abrir o painel e mantém uma cópia embutida como reserva, então ela continua
funcionando offline e mesmo que este repositório esteja fora do ar.

## A regra que decide o que entra

O Python que a AURORA embarca é um build MinGW do MSYS2, não o CPython oficial
da python.org. Isso tem uma consequência que não se contorna:

| Tipo de biblioteca | Roda? |
|---|---|
| Python pura — wheel `*-none-any.whl` | **Sim.** É só descompactar. |
| Com extensão em C — `cp312-win_amd64` e afins | **Não.** Nunca. |

O motivo é concreto: o `.pyd` compilado procura a `python312.dll` da Microsoft,
que não existe no nosso build (ele expõe `libpython3.12.dll`). O erro é
`DLL load failed`, e renomear o arquivo não resolve — a incompatibilidade é no
vínculo binário.

Por isso o gerador do catálogo **recusa automaticamente** qualquer biblioteca sem
wheel pura. Não é uma questão de preferência: é a única classe que carrega.

NumPy, SciPy, Matplotlib e pandas aparecem no catálogo como entradas
informativas, sem arquivos, para o painel poder mostrá-las e explicar por que
ainda não instalam, em vez de fingir que não existem.

## Estrutura do catálogo

```jsonc
{
  "schemaVersion": 1,          // a AURORA recusa formato que não sabe ler
  "python": {
    "validatedAgainst": "3.12",
    "abiTag": "cp312",
    "platform": "mingw_x86_64_msvcrt_gnu"
  },
  "categories": { "hdl": { "pt": "Verificacao", "en": "Verification" } },
  "libraries": [
    {
      "id": "plotly",
      "name": "Plotly",
      "version": "6.9.0",
      "kind": "pure",           // "pure" instala; "compiled" é informativa
      "category": "viz",
      "icon": "plotly",         // nome de um símbolo que a AURORA já tem
      "homepage": "https://plotly.com/python/",
      "license": "MIT",
      "summary": { "pt": "...", "en": "..." },
      "uses":    { "pt": ["..."], "en": ["..."] },
      "downloadSize": 10446849,
      "wheels": [               // o fecho COMPLETO de dependências, fixado
        { "name": "plotly", "version": "6.9.0",
          "filename": "plotly-6.9.0-py3-none-any.whl",
          "url": "https://files.pythonhosted.org/...",
          "sha256": "...", "size": 9948160 }
      ]
    }
  ]
}
```

Não há resolvedor de dependências, e isso é proposital: sem `pip` no runtime
embarcado não existe resolvedor, então o catálogo entrega o fecho pronto e o
instalador apenas obedece à lista. Resultado igual em todas as máquinas.

## O que a lista consegue e o que não consegue mudar

Vale ser explícito sobre o limite, porque ele orienta o que faz sentido colocar
aqui.

A lista carrega **dado**: nome, descrição, casos de uso, versão, hash,
categoria. Tudo isso chega pela rede e aparece na AURORA sem release nenhum.

A lista **não** carrega código nem desenho. As integrações nativas — botões,
painéis e comandos que a AURORA oferece para uma biblioteca específica — são
código da IDE, e o desenho dos ícones vive nos recursos dela. Uma biblioteca
acrescentada aqui hoje fica plenamente utilizável (instala, importa, roda no
testbench), mas os atalhos na interface só chegam na próxima versão da AURORA.

Por isso o campo `icon` guarda o **nome** de um símbolo, não o desenho. A AURORA
traz um conjunto de símbolos de categoria e as marcas das bibliotecas mais
conhecidas; um nome que ela não reconheça cai no símbolo genérico, sem quebrar
nada. O mesmo vale para `category`.

## Como alterar a lista

O gerador vive na AURORA, porque precisa consultar a PyPI e conferir a regra da
wheel pura. O ciclo é:

```bash
# no repositório da AURORA
node scripts/gen-pylib-catalog.js      # edita CURATED em scripts/gen-pylib-catalog.js
npx vitest run tests/unit/pylibs.test.js

# publica aqui
cp resources/pylib-catalog.json ../aurora-pylibs/catalog.json
cd ../aurora-pylibs && git commit -am "libs: ..." && git push
```

A cópia embutida na AURORA (`resources/pylib-catalog.json`) deve ser commitada
junto: é ela que sustenta o uso offline e o primeiro carregamento.

O passo a passo completo, incluindo como conferir se uma biblioteca é pura antes
de tentar adicioná-la, está em [`docs/PASSO-A-PASSO.txt`](docs/PASSO-A-PASSO.txt).

## Compatibilidade

`schemaVersion` protege quem não atualizou. Se este arquivo passar a usar um
formato mais novo do que a AURORA instalada sabe ler, ela ignora o catálogo
remoto, usa a cópia embutida e avisa que existe uma atualização — em vez de
interpretar errado e quebrar.

Na prática: **subir o `schemaVersion` só quando a mudança for incompatível.**
Acrescentar biblioteca, mudar descrição ou bumpar versão não mexe nele.

## Licença

O catálogo (este repositório) é MIT. As bibliotecas listadas pertencem aos seus
autores e mantêm suas próprias licenças — o campo `license` de cada entrada traz
a declarada na PyPI. Nada aqui redistribui código de terceiros: a AURORA baixa
direto da PyPI.
