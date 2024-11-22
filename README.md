# Conversor RPN/Infix

Este é um programa simples em C que converte uma expressão matemática de Notação Polonesa Reversa (RPN) para Infixa (ou vice-versa).

Esta biblioteca foi escrita em C99 e utiliza a suíte de testes [Check](https://libcheck.github.io/check). Suas únicas dependências são Check, GNU make (para compilar) e GNU GCC (para compilar o código). Além disso, pode ser necessário instalar o `libsubunit-dev` para o Check, dependendo do sistema operacional — se você receber o erro "Cannot find '-lsubunit'" do GCC, essa seria a maneira de corrigir.

Para compilar o script, execute `make convert`. Para compilar o script de testes, execute `make tests`. Os executáveis serão `convert` e `convert_tests`, respectivamente. `make all` irá compilar ambos os scripts, e `make clean` irá removê-los, bem como quaisquer artefatos de compilação.

`convert` não aceita argumentos, mas irá solicitar ao usuário uma expressão inicial com no máximo 500 caracteres e o tipo de conversão. A expressão inicial deve consistir em operandos na forma de letras minúsculas, operadores válidos (`^`, `/`, `*`, `-`, `+`) e parênteses (`(` e `)` apenas).

`convert_tests` executará todos os testes unitários contidos no diretório `test/`, e também não aceita entrada.

## API da Biblioteca

O diretório `src/` contém o código-fonte deste programa:
- `main.c`: permite ao usuário inserir uma expressão e o tipo de conversão, e receber o resultado
- `rpn_to_infix.c`: contém o algoritmo de conversão de RPN para Infix, além de várias funções auxiliares
- `infix_to_rpn.c`: contém o algoritmo de conversão de Infix para RPN, além de várias funções auxiliares
- `utilities.c`: contém funções auxiliares compartilhadas pelo programa (principalmente relacionadas à identificação de caracteres), além dos métodos `append` e `pop` para a pilha.

Os arquivos de cabeçalho estão contidos no diretório `include/`, e os testes unitários no diretório `test/`, nomeados de acordo com o que testam.

## Métodos de Conversão

Esta seção contém minha solução para o problema. Elaborei os algoritmos e regras sintáticas usando papel e lápis, sem consulta a soluções online de qualquer tipo. Quando obtive um conjunto de regras que parecia funcionar, implementei-as e segui o seguinte ciclo:

1. Tentei quebrar o que tinha com entradas complexas/estranhas/malformadas na forma de um teste.
2. Corrigi o código, usando sites de referência de C99 conforme necessário.
3. Repeti os passos 1 e 2 até não conseguir pensar em novas maneiras de quebrar o código.

Além disso, observe que uso `variável` e `operando` de forma intercambiável ao longo do restante deste README.

### Verificação Pré-conversão

Antes de tentar uma conversão, o programa verifica se a expressão de entrada é válida ou não. Se a expressão for inválida, o programa lançará um erro e será encerrado.

#### RPN

Uma expressão RPN é considerada válida se:

1. A expressão contém EXATAMENTE `n` variáveis e `n-1` operadores, já que todos os operadores suportados exigem 2 operandos para serem executados com sucesso (portanto, `ab`, `+++` e `-b` são inválidos).
2. A expressão contém apenas variáveis e operadores (sem parênteses, outros símbolos/letras, etc.).
3. Em nenhum momento, desde o início da expressão, deve haver o mesmo número de variáveis e operadores (por exemplo, `a+b` é inválido porque no `+`, não há operadores suficientes "atrás" dele para executar com sucesso. O mesmo vale para `ab+c--d`, etc.).

#### Infix

Uma expressão Infix é considerada válida se:

1. A expressão contém EXATAMENTE `n` variáveis e `n-1` operadores, pelos mesmos motivos da RPN.
2. Há um número igual de `(` e `)` na expressão.
3. A expressão contém apenas variáveis, operadores e parênteses.
4. Cada substring, definida como uma seção da expressão contida entre parênteses, também é válida, dados os critérios acima. Por exemplo, dada a expressão `((a+b)+c)+d`, `(a+b)` e `((a+b)+c)` DEVEM ser válidas para que a expressão completa seja válida. Isso permite detectar expressões como `(a+)b`, que satisfazem as três primeiras regras acima, mas são claramente inválidas.

### RPN para Infix

O script primeiro cria uma pilha de tamanho 50, com cada elemento da pilha capaz de armazenar uma expressão RPN de até 1000 caracteres (qualquer tentativa de ultrapassar o limite ou esvaziar a pilha resultará em um erro e encerrará o programa).

Para cada caractere na expressão RPN:
- Se o caractere for uma variável, adicione-o à pilha.
- Se o caractere for um operador:
    - Remova os dois elementos no topo da pilha (se houver menos de dois elementos na pilha, reporte um erro e encerre).
    - Combine o resultado das duas remoções com o operador, envolvendo o resultado entre parênteses.
    - Adicione esta nova expressão à pilha.

No final do loop e, assumindo que a expressão RPN era válida, o resultado será uma única expressão na pilha, que será a expressão Infix. Essa expressão é retornada e exibida na tela. Se houver mais de uma expressão restante na pilha, houve um erro sintático na expressão inicial, e um erro será reportado.

Como exemplo, considere a expressão `vw/x^yz-*`. A execução do loop seria como a tabela abaixo:

| Caractere |         Pilha          |
|-----------|------------------------|
|     v     |   v                    |
|     w     |   v, w                 |
|     /     |   (v/w)                |
|     x     |   (v/w), x             |
|     ^     |   ((v/w)^x)            |
|     y     |   ((v/w)^x), y         |
|     z     |   ((v/w)^x), y, z      |
|     -     |   ((v/w)^x), (y-z)     |
|     *     |   (((v/w)^x)*(y-z))    |

Portanto, `(((v/w)^x)*(y-z))` seria o que é retornado ao usuário. Observe que a expressão final está envolvida em um conjunto de parênteses. Esses parênteses são tecnicamente redundantes, mas a expressão ainda é válida.

### Infix para RPN

O script cria duas pilhas, uma para variáveis e outra para símbolos (os cinco operadores válidos mais `(` e `)`). Ambas podem armazenar até 1000 caracteres (qualquer tentativa de ultrapassar o limite ou esvaziar a pilha resultará em um erro e encerrará o programa).

Para cada caractere na expressão Infix:
- Se o caractere for uma variável, adicione-o à pilha de variáveis.
    
- Se o caractere for um símbolo:
    - Se o caractere for `)`:
        - A partir do topo da pilha de operadores, remova todos os operadores e adicione-os à pilha de variáveis.
        - Se este loop alcançar um `(`, remova-o e encerre o loop.
        - Se este loop alcançar o fim da pilha de operadores sem encontrar um `(`, houve parênteses desbalanceados na expressão, então lance um erro.
    - Se o caractere `c1` for um operador e a pilha de símbolos não estiver vazia:
        - Se o símbolo no topo da pilha de símbolos `c2` for um operador E ou (`c1` for associativo à direita e tiver maior precedência que `c2`) OU (`c1` for associativo à esquerda e tiver precedência maior ou igual a `c2`), remova `c2` da pilha de símbolos e adicione-o à pilha de variáveis. 
        - Repita para cada operador no topo da pilha de símbolos, encerrando o loop se a condição acima não for verdadeira.
        - Adicione `c1` à pilha de símbolos.
    - Caso contrário, adicione `c1` à pilha de símbolos.
- Caso contrário, este caractere é inválido — lance um erro e encerre o programa.

No final do loop, todos os símbolos restantes na pilha de símbolos são adicionados à pilha de variáveis, e o conteúdo da pilha de variáveis contém a expressão RPN final. Se houver um parêntese na pilha de símbolos ou variáveis, então temos parênteses desbalanceados, e o programa lança um erro.

Como exemplo, considere a expressão `((v/w)^x)*(y-z)` do exemplo anterior. A execução do loop seria como a tabela abaixo:

| Caractere |      Pilha de Variáveis      | Pilha de Operadores |
|-----------|------------------------------|---------------------|
|     (     |                              | (                   |
|     (     |                              | (, (                |
|     v     | v                            | (, (                |
|     /     | v                            | (, (, /             |
|     w     | v, w                         | (, (, /             |
|    
