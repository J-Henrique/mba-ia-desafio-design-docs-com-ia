# Rastreabilidade Obrigatória

Toda informação registrada nos documentos (PRD, RFC, FDD, ADRs, Tracker, README) deve ser **rastreável** à transcrição (`TRANSCRICAO.md`) ou ao código fonte da aplicação.

## Regras

1. **Não inventar.** Requisitos, decisões ou restrições sem origem identificável são proibidos.
2. **Citar origem.** Cada afirmação nos documentos deve referenciar o trecho da transcrição ou do código que a justifica.
3. **Tracker como fonte da verdade.** O Tracker de rastreabilidade é o artefato que liga cada item à sua origem — ele deve ser mantido consistente com os demais documentos.
4. **Revisão de consistência.** Antes de finalizar qualquer documento, verificar se todo conteúdo tem origem rastreável.
5. **Importante:** NADA pode ser adicionado que não tenha sido discutido na reunião. Isso inclui:
   - Alternativas consideradas em ADRs — só as que apareceram na transcrição
   - Ferramentas, frameworks, bibliotecas não mencionadas na call
   - Números, prazos, métricas que não tenham origem explícita na transcrição ou no código
   
   Se algo não está na transcrição nem no código, não entra. Antes de incluir qualquer alternativa/opção, verificar se o texto da transcrição a menciona. Este erro já ocorreu em ADR-006 (framework externo inventado) e foi corrigido.
6. **Sinalização de conteúdo plausível:** conteúdo "plausível" (não discutido na reunião, mas razoável de mercado) **pode** ser incluído **desde que explicitamente sinalizado como tal** — ex: `> Plausível (não discutido na reunião): ...`. Requisitos, decisões e restrições continuam proibidos sem origem; a sinalização só se aplica a alternativas/opções auxiliares. O RFC pede "alternativas reais discutidas" — ali, plausível não entra.
