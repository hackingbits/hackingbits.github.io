---
layout: post
title: O menor do mundo? Yeah? So Beat the Bits!
date: '2013-03-25T08:35:00.000-03:00'
author: geyslan
categories: blog
excerpt: ""
tags: [shellcode, linux, code, assembly, c, hacking, portuguese]
share: true
modified: '2013-08-20T15:24:46.146-03:00'
thumbnail: http://3.bp.blogspot.com/-WNiLpZTzqOQ/UU4c21RajkI/AAAAAAAAAIQ/R4G1XQRqsPw/s72-c/Registers.png
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-3414477427392965127
blogger_orig_url: http://www.hackingbits.com/2013/03/o-menor-do-mundo-yeah-so-beat-bits.html
---

*/\*<br>
Off-Topic: Está disponível no [Shell-Storm](http://shell-storm.org/) versão do
[Shell Bind TCP](http://shell-storm.org/shellcode/files/shellcode-835.php)
usando o método GetPC (GetEIP). Os demais shellcodes apresentados neste post
também já foram disponibilizados no mesmo repositório. Tks again, Salwan.<br>
\*/*

<!--more-->

Enquanto que em muita área por aí o que importa é ter ou fazer algo grande,
quando falamos em shellcodes sempre os queremos minúsculos, correto? E mesmo que
isso não seja tão importante para as mais recentes técnicas de exploração, você
não gostaria que o seu fosse menor? =D

Bom, como sou teimoso, aproveitei o tempo de viagem de onde estou trabalhando
até minha casa (10h de ônibus) para brincar um pouco com as versões anteriores
que disponibilizei e consequentemente aprender mais. Meu notebook, graças ao
[Jupiter](http://jupiter.sourceforge.net/downloads.html), aguentou 7h de pdfs,
chrome (em offline), editores, compilação e muito debugging.

O que fazer para se diminuir o tamanho de um shellcode?

Com a questão em mente, condensei nas regras abaixo o que aprendi.

##Regra n. 01 - Certificar-se da "pureza" dos registers antes de serem preenchidos com valores menores que o da arquitetura

Num ambiente de desenvolvimento e exploração, ao se testar um shellcode
projetado, os registers se apresentam sem lixo, o que pode mascarar o seu real
funcionamento.

Temos que ter em mente que um shellcode é um pedaço de código injetado em um
programa já em execução com suas STACK e registers já em utilização. Se
começarmos a simplesmente utilizar esses registers ou a STACK sem as devidas
providências o shellcode não vai servir ao seu propósito.

Na arquitetura 32 bits, se eu movimentar (não gosto de usar o termo movimentar;
mesmo a instrução se chamando MOV o que ela faz, em verdade, é copiar) um valor
para um register [mov eax, 10], a instrução preenche totalmente o EAX com o
valor imediato.

Se antes da cópia EAX tiver 0xffffffff, após ele ficará com 0x0000000a:

{% highlight console %}
b8 0a 00 00 00        mov    eax,0xa
{% endhighlight %}

Até então sem problemas, certo? Não, para um shellcode! O opcode B8 que
movimenta o valor imediato 0xa (10) preenche todos os 32 bits de EAX, e você
deve se recordar que não podemos ter null bytes para que o payload seja
funcional.

Contornando a situação com [mov al, 10]:

{% highlight console %}
b0 0a                 mov    al,0xa
{% endhighlight %}

Like a charm? Copiamos agora apenas o byte necessário *0xa* para o register
**AL** de 8 bits.

{% include imagehb.html url="http://3.bp.blogspot.com/-WNiLpZTzqOQ/UU4c21RajkI/AAAAAAAAAIQ/R4G1XQRqsPw/s1600/Registers.png" caption="Registers" %}

Contudo, se EAX já estiver, hipoteticamente, com 0xffffffff, ao copiarmos apenas
para AL, o valor final será 0xffffff0a (-246). Qualquer syscall ao usar EAX
retornará erro.

###Dando a volta por cima


Eu estava usando um truque com a instrução
[CDQ](http://faydoc.tripod.com/cpu/cdq.htm) - convert double to quad - para
zerar EDX, após preparar EAX com algum valor inicial, ex: PUSH 10 (6A, 0A), POP
EAX (58). A CDQ é de apenas um opcode (99), por isso me encantou. Em suma, com
ela eu podia despoluir EDX e já deixar EAX preparada com apenas 3 bytes.

Sendo que o zero é essencial para o preenchimento dos argumentos das syscalls
mas não pode estar presente no shellcode, essa técnica permite termos o zero em
EDX para utilização posterior.

*E eu não poderia fazer isso usando XOR? Sim, mas a custo de mais um byte.*

{% highlight console %}
31 d2                 xor    edx,edx
...
99                    cdq
{% endhighlight %}

E para limpar os demais registers necessários no início do shellcode sempre
serão utilizados dois opcodes, seja com a técnica do XOR,  com a do PUSH reg,
POP reg ou com a do MOV reg, reg.

{% highlight console %}
31 d2                 xor    edx,edx
...
52                    push   edx     #que já será zero após o cdq
5b                    pop    ebx
...
89 dd                 mov    ebp,ebx
{% endhighlight %}

De tanto garimpar, descobri um truque com o uso da instrução
[MUL](http://faydoc.tripod.com/cpu/mul.htm) - unsigned multiply - que permite
despoluir três registers de uma só vez (EAX, EBX [poderia ser outro: ECX, ESI, ...]
e EDX).

{% highlight console %}
31 db                 xor    ebx,ebx
f7 e3                 mul    ebx
{% endhighlight %}

Ele gera um byte a mais, entretanto um a menos se usarmos um XOR, para zerar EBX
logo após o uso do CDQ.

Temos também o mágico [XCHG](http://faydoc.tripod.com/cpu/xchg.htm) que permuta
os valores de dois registers a custo de apenas um byte se um dos envolvidos na
troca for o EAX; qualquer outro tipo de permuta terá dois opcodes.

{% highlight console %}
95                    xchg   ebp,eax
96                    xchg   esi,eax
87 d9                 xchg   ecx,ebx
{% endhighlight %}

De toda sorte, essas formas (CDQ, MUL, MOV, XOR e XCHG) são boas e devem, se
possível, ser utilizadas em conjunto. O sucesso na redução do tamanho irá
depender de quais registers serão necessários às syscalls do shellcode e de que
valores já estarão contidos neles, dando vantagem a uma forma em detrimento às
demais.

##Regra n. 02 - Reutilizar dados já inseridos na STACK

Eu estava simplesmente fazendo PUSH de todos os argumentos necessários a cada
syscall, desprezando o que eu já havia inserido na STACK. Obviamente os
primeiros valores a serem utilizados precisam ser inseridos por nós mesmos, pois
não sabemos o que há na pilha (o contrário pode ser dito se se tratar de uma
aplicação em específico, cujo exploit fora desenvolvido por nós mesmos).

*Socket*
{% highlight console %}
52                    push   edx
53                    push   ebx
6a 02                 push   0x2
89 e1                 mov    ecx,esp
{% endhighlight %}

*Bind*

{% highlight console %}
b3 02                 mov    bl,0x2
52                    push   edx
66 68 2b 67           pushw  0x672b
66 6a 02              pushw  0x2
89 e1                 mov    ecx,esp
{% endhighlight %}

No exemplo acima, sem reutilizar os dados da STACK, ao criar a estrutura
sockaddr_in para fazer o Bind, são gerados 12 bytes. Todavia, vejam a diferença
fazendo uso dos dados já existentes.

{% highlight console %}
5b                    pop    ebx
5e                    pop    esi
52                    push   edx
66 68 2b 67           pushw  0x672b
{% endhighlight %}

Fantasticamente geramos apenas 7 bytes (cinco a menos). É importante ressaltar
que a instrução POP apenas copia o valor da pilha para algum destino,
modificando o ponteiro (ESP) do topo da pilha e deixando o valor intacto na
memória, contrariamente à PUSH que o substitui. Atente para o fato de que ECX já
contém o ponteiro para os dados que estamos utilizando, portanto, não o
alteramos.

Se algum dado distante na STACK precisar ser modificado para reutilização
correta dos demais, tiramos proveito fazendo a modificação com a MOV e um
ponteiro, gerando apenas 3 bytes.

{% highlight console %}
89 51 04              mov    DWORD PTR [ecx+0x4],edx
{% endhighlight %}

Mais uma vez, devemos ponderar qual método é mais eficaz em cada caso.

##Regra n. 03 - Usar instruções com menor número de opcodes

Mesmo esta regra sendo óbvia e estar implícita nas anteriores, faço questão de
exemplificar com algumas instruções, para que haja melhor entendimento.

{% highlight console %}
43                    inc    ebx
b3 02                 mov    bl,0x2
{% endhighlight %}

No caso acima, se EBX já contivesse 1, mais inteligente seria apenas o
incrementarmos com INC, em vez de o atribuirmos 2 com MOV. Abaixo, segue exemplo
de redução usando a [LEA](http://faydoc.tripod.com/cpu/lea.htm) - load effective
address - para uso aritmético com vários operandos em vez da
[ADD](http://faydoc.tripod.com/cpu/add.htm) (tks [Pedro
Fausto](https://plus.google.com/106164431356595417628/)).

{% highlight console %}
01 d8                 add    eax,ebx
83 c0 02              add    eax,0x2

8d 44 18 02           lea    eax,[eax+ebx*1+0x2]
{% endhighlight %}

##Regra n. 04 - GetPC (GetEIP) - Adendo

Esqueci-me de mencionar, na postagem original, a técnica GetPC que utilizei como
demonstração no shellcode [Shell Bind TCP -
GetPC](http://shell-storm.org/shellcode/files/shellcode-835.php) (github:
[assembly](https://github.com/geyslan/SLAE/blob/master/improvements/shell_bind_tcp_getpc.asm),
[fluxograma](https://raw.github.com/geyslan/SLAE/master/improvements/shell_bind_tcp_getpc.png)).

Se o shellcode tiver um tamanho considerável e houver uma certa quantidade de
instruções que se repetem, esse método é válido para se diminuir o payload
usando as CALL e RET.

Analisem os códigos! Vale à pena.

##Os menores do mundo? Não sei, diga-me!

O resultado de tanto mexido foi o engendramento dos três, aparentemente, menores
shellcodes do mundo nas suas respectivas especificidades.

[Shellcode
Improvements](https://github.com/geyslan/SLAE/tree/master/improvements) (github
com os concernentes fluxogramas e shellcode launchers)

[Tiny Shell Bind TCP - 73 bytes](https://github.com/geyslan/SLAE/blob/master/improvements/tiny_shell_bind_tcp.asm)<br>
[Tiny Shell Bind TCP Random Port - 57 bytes](https://github.com/geyslan/SLAE/blob/master/improvements/tiny_shell_bind_tcp_random_port.asm)<br>
[Tiny Shell Reverse TCP - 67 bytes](https://github.com/geyslan/SLAE/blob/master/improvements/tiny_shell_reverse_tcp.asm)

##Eu desafio vocês!

Pensando nisso estou lançando um concurso para redução dos shellcodes
apresentados. Para concorrer basta criar um shellcode dos tipos acima com menor
tamanho em bytes e enviar para [geyslan@gmail.com](mailto:geyslan@gmail.com).
Para redução podem ser utilizados, se preferirem, os que disponibilizei.

###Premiação

Aprendizado! E menos bytes! ;D

Vemo-nos em breve!

##Mais Informações

[aiuto: Java, C++ & ASM Ref](https://play.google.com/store/apps/details?id=in.nishitp.aiuto)<br>
[Basic Assembly Language (x86)](https://play.google.com/store/apps/details?id=com.mrdroids.nasm)<br>
[Shell-Storm](http://shell-storm.org/)<br>
[Intel 64 and IA-32 Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)<br>
