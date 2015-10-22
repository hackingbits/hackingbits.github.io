---
layout: post
title: "(Des)construindo Software"
date: '2013-01-23T20:10:00.001-03:00'
author: geyslan
tags: [linux, code, c, elf, reverse engineering, portuguese]
modified_time: '2013-08-20T14:10:38.486-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-4729046667090812298
blogger_orig_url: http://www.hackingbits.com/2013/01/desconstruindo-software.html
---

*“A Engenharia Reversa é o processo de extrair conhecimento ou design de algo
feito pelo homem (…) O Software é uma das mais complexas e intrigantes
tecnologias dentre nós nos dias de hoje. Sua engenharia reversa se trata de
abrir o programa e olhá-lo por dentro. Claro, nós não precisamos de nenhuma
chave de fenda nesta jornada. Da mesma forma que a engenharia de software, seu
reverso é puramente um processo virtual envolvendo apenas um processador e a
mente humana.”* **Reversing – Secrets of Reverse Engineering – Eldad Eilam**.

<!--more-->

Antes de nos debruçarmos na reversão em si, é importante sabermos como se dá o
processo de engendramento do software binário.

Para a sua construção, faz-se necessária (a não ser que vocês sejam masoquistas
e queiram construir em código de máquina) uma linguagem de programação. Essa se
traduz como um método padronizado para informar instruções a um processador.
Cada linguagem detém sua própria padronização assim como seu nível de abstração
(baixo, médio e alto).

Na sequência, são destacados os procedimentos demandados para a transformação do
código escrito em um bloco de instruções “entendível” aos olhos do computador.

##Construção de Software

A construção ou compilação é um processo de vários estágios que envolve várias
ferramentas. No nosso caso, as ferramentas usadas serão o front-end **gcc** (GNU
Compiler), o **cc1** (C Compiler), o **as** (GNU Assembler), o **collect2** (ld
wrapper), e o **ld** (GNU Linker). O conjunto completo de ferramentas
utilizadas no processo de compilação é chamado toolchain, que, quase que por
padrão, já vem instalado nas distribuições GNU/Linux. A criação de toolchains
específicas são primordiais para o cross-compiling, mas esse assunto fica para
um post futuro.

Durante uma invocação do GCC a sequência de comandos executadas percorre os
seguintes estágios:

- pré-processamento (expansão de macros)
- compilação (do código fonte para linguagem assembly)
- assembler (de linguagem assembly para código de máquina)
- linking (criação do binário final)

{% include imagehb.html url="https://raw.github.com/geyslan/hb/master/desconstruindo/GCC.Stages.png" caption="Estágios da Compilação" %}

Criemos como exemplo um programa minimalista em C com o famoso Hello World.

{% highlight console %}
$ mkdir construindo; cd construindo
$ echo -e "#include <stdio.h>\nint main() { printf(\"Hello World\\\n\"); return 0;}" > programa1.c
{% endhighlight %}

Para esta compilação, usemos o gcc.

{% highlight console %}
$ gcc programa1.c -o programa1
{% endhighlight %}

Ao rodar o gcc, as etapas de pré-processamento, compilação, assembly e linking
são efetuadas uma após a outra de forma transparente, até que o binário ELF
**programa1** seja criado.

{% highlight console %}
$ ./programa1
Hello World
{% endhighlight %}

Em seguida, para melhor entendimento, percorreremos cada fase manualmente.

##Pré-processamento e Compilação

[Pré-processamento](###http://pt.wikipedia.org/wiki/Pr%C3%A9-processador)

Em versões passadas do gcc, o pré-processamento era feito como um estágio em
separado através do cpp (C Preprocessor).

{% highlight console %}
$ cpp programa1.c -o programa1.i
{% endhighlight %}

A saída era um arquivo com as macros expandidas e declarações de arquivos header
incluídas. O pré-processador, em verdade, traduzia a abstração média da
linguagem C (na qual programamos) para uma reconhecível à próxima etapa da
compilação.

{% highlight c %}
# 1 "programa1.c"
# 1 "<command-line>"
# 1 "programa1.c"
# 1 "/usr/include/stdio.h" 1 3 4
# 28 "/usr/include/stdio.h" 3 4
# 1 "/usr/include/features.h" 1 3 4
...
extern void funlockfile (FILE *__stream) __attribute__ ((__nothrow__ , __leaf__));
# 940 "/usr/include/stdio.h" 3 4
# 2 "programa1.c" 2
int main(void) { printf("Hello World\n"); return 0;}
{% endhighlight %} [//]: # (fix-highlight*)

Esse primeiro estágio, atualmente, é incorporado pelo compilador
[cc1](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)
quando usado com opções default; ou seja, geralmente é omitido.

{% highlight console %}
$ gcc programa1.c -o programa1 -v
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -quiet -v -imultiarch x86_64-linux-gnu programa1.c -quiet -dumpbase programa1.c -mtune=generic -march=x86-64 -auxbase programa1 -version -fstack-protector -o /tmp/ccZvPRPS.s
...
{% endhighlight %}

Como pode ser visto no modo verboso (detalhado), o cc1 efetua o
pré-processamento e compilação, gerando diretamente o arquivo assembly. De toda
forma podemos fazer com que o cc1 seja invocado duas vezes, com o uso da opção
**-no-integrated-cpp**; primeiramente para o pré-processamento, gerando o arquivo
expandido (\*.i) e posteriormente para a compilação, gerando o arquivo para
montagem (\*.s). Tal procedimento é útil quando se pretende utilizar um
pré-processador alternativo, por exemplo.

{% highlight console %}
# gcc programa1.c -o programa1 -v -no-integrated-cpp
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu programa1.c -mtune=generic -march=x86-64 -fstack-protector -o /tmp/ccjNwBsL.i
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -fpreprocessed /tmp/ccjNwBsL.i -quiet -dumpbase programa1.c -mtune=generic -march=x86-64 -auxbase programa1 -version -fstack-protector -o /tmp/cc0B4fmf.s
...
{% endhighlight %}

Podemos, também, interromper o processo do gcc, invocando o cc1 apenas uma vez
com o intuito de gerarmos o arquivo (\*.i) pré-processado.

{% highlight console %}
# gcc programa1.c -o programa1.i -v -E
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu programa1.c -o programa1.i -mtune=generic -march=x86-64 -fstack-protector
...
{% endhighlight %}

Para isso utilizamos a opção **-E**.

[Compilação](http://en.wikipedia.org/wiki/Compiler)

O objetivo principal do uso do cc1, através ou não do arquivo expandido
(programa1.i), é gerar o respectivo código assembly, linguagem de baixo nível.

{% highlight console %}
$ gcc programa1.i -o programa1.s -v -S
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -fpreprocessed programa1.i -quiet -dumpbase programa1.i -mtune=generic -march=x86-64 -auxbase-strip programa1.s -version -o programa1.s -fstack-protector
...

$ gcc programa1.c -o programa1.s -v -S
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/cc1 -quiet -v -imultiarch x86_64-linux-gnu programa1.c -quiet -dumpbase programa1.c -mtune=generic -march=x86-64 -auxbase-strip programa1.s -version -o programa1.s -fstack-protector
...
{% endhighlight %}

A opção **-S**, instrui o gcc a apenas converter o código C, pré-processado ou
não, para a linguagem de montagem (programa1.s).

{% highlight asm %}
.file "programa1.c"
.section .rodata
.LC0:
.string "Hello World"
.text
.globl main
.type main, @function
main:
.LFB0:
.cfi_startproc
pushq %rbp
.cfi_def_cfa_offset 16
.cfi_offset 6, -16
movq %rsp, %rbp
.cfi_def_cfa_register 6
movl $.LC0, %edi
call puts
movl $0, %eax
popq %rbp
.cfi_def_cfa 7, 8
ret
.cfi_endproc
.LFE0:
.size main, .-main
.ident "GCC: (Ubuntu/Linaro 4.7.2-2ubuntu1) 4.7.2"
.section .note.GNU-stack,"",@progbits
{% endhighlight %}

Atenção! O código assembly pode estar diferente do gerado em seu computador.
Essa divergência pode ter sido ocasionada por vários motivos: versão do GCC;
arquitetura utilizada; flags de compilação.

Para gerar o respectivo código em 32 bits, basta utilizar a opção **-m32**.

{% highlight console %}
$ gcc programa1.i -S -m32
{% endhighlight %}

ou

{% highlight console %}
$ gcc programa1.c -S -m32
{% endhighlight %}

Vejam a diferença:

{% highlight asm %}
.LFB0:
.cfi_startproc
pushl %ebp                   # <-
.cfi_def_cfa_offset 8        # <-
.cfi_offset 5, -8            # <-
movl %esp, %ebp              # <-
.cfi_def_cfa_register 5      # <-
andl $-16, %esp              # <-
subl $16, %esp               # <-
movl $.LC0, (%esp)call puts  # <-
movl $0, %eax
leave                        # <-
.cfi_restore 5               # <-
cfi_def_cfa 4, 4ret          # <-
.cfi_endproc
{% endhighlight %}

##Assembler (Montador)

Na [montagem](http://en.wikipedia.org/wiki/Assembler_(computing)#Assembler), o código assembly é convertido nas suas instruções correlatas em código de máquina.
 O resultado é um arquivo-objeto (object file).

{% highlight console %}
$ gcc programa1.s -o programa1.o -v -c
...
as -v --64 -o programa1.o programa1.s
...

$ gcc programa1.s -o programa1.o -v -m32 -c
...
as -v --32 -o programa1.o programa1.s
...
{% endhighlight %}

A opção **-c** faz com o gcc apenas compile ou monte o arquivo fonte, mas não o
linque.

Como visto acima, o [as](http://en.wikipedia.org/wiki/GNU_Assembler)
faz a montagem do binário usando o arquivo assembly (programa1.s). Ao rodarmos
o file contra o arquivo-objeto construído, podemos verificar a estrutura ELF.

{% highlight console %}
$ file programa1.o
programa1.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped

programa1.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
{% endhighlight %}

Entretanto, como o processo de compilação ainda não foi finalizado, ao tentarmos
executá-lo recebemos a mensagem de que é impossível executar o arquivo binário.
Ao usarmos o ltrace (ferramenta de análise de chamadas à bibliotecas) recebemos
a mensagem **./programa1.o is not an ELF executable nor shared library**. O
assembler construiu o bloco de instruções binárias para a arquitetura, contudo
não definiu os endereços referentes às funções externas, assim como não relocou
o binário para a correta execução.

###Linking

Até então o arquivo-objeto montado não sabe para onde olhar (endereço das
funções necessárias ao correto funcionamento). O Linking ou lincagem é, neste
caso, o procedimento de aglutinação de arquivos-objeto, resolução de símbolos e
relocação das seções e respectivos endereços do binário outrora gerado.

Vejamos.

{% highlight console %}
$ gcc programa1.o -o programa1 -v
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/collect2 --sysroot=/ --build-id --no-add-needed --as-needed --eh-frame-hdr -m elf_x86_64 --hash-style=gnu -dynamic-linker /lib64/ld-linux-x86-64.so.2 -z relro -o programa1 /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.7/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/4.7 -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../.. programa1.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/4.7/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../x86_64-linux-gnu/crtn.o
...
{% endhighlight %}

ou

{% highlight console %}
$ gcc programa1.o -o programa1 -v -m32
...
/usr/lib/gcc/x86_64-linux-gnu/4.7/collect2 --sysroot=/ --build-id --no-add-needed --as-needed --eh-frame-hdr -m elf_i386 --hash-style=gnu -dynamic-linker /lib/ld-linux.so.2 -z relro -o programa1 /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../../lib32/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../../lib32/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.7/32/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/4.7/32 -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../../i386-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../../../lib32 -L/lib/i386-linux-gnu -L/lib/../lib32 -L/usr/lib/i386-linux-gnu -L/usr/lib/../lib32 -L/usr/lib/gcc/x86_64-linux-gnu/4.7 -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../../i386-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.7/../../.. -L/lib/i386-linux-gnu -L/usr/lib/i386-linux-gnu programa1.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/4.7/32/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.7/../../../../lib32/crtn.o
...
{% endhighlight %}

O que o gcc realmente faz é invocar o [collect2](http://gcc.gnu.org/onlinedocs/gccint/Collect2.html) (wrapper para o
[ld](http://en.wikipedia.org/wiki/GNU_linker) (binutils)) informando todos os
arquivos-objeto que devem ser lincados.

É verdade que o ld pode ser utilizado diretamente; de toda sorte, o gcc/collect2
já informa todas as libs necessárias para o linking, o que facilita bastante.

Após terminado o processo, é possível verificar que o executável está
devidamente lincado.

{% highlight console %}
$ file programa1
programa1: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=0xa0c...bf73, not stripped

programa1: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=0xe07...8df3, not stripped
{% endhighlight %}

E, finalmente, executá-lo.

{% highlight console %}
$ ./programa1
Hello World
{% endhighlight %}

##Conclusão

Pudemos, neste post, acompanhar de forma breve o processo de construção de um
executável ELF, exemplificando os respectivos procedimentos no ambiente
GNU/Linux através do GCC. No próximo, iniciaremos uma abordagem prática da
engenharia reversa que possibilitará um entendimento objetivo da estrutura ELF,
assim como do método e das ferramentas utilizadas na reversão.

Até lá!

##Mais Informações
[Reversing – Secrets of Reverse Engineering – Eldad Eilam](http://www.amazon.com/Reversing-Secrets-Engineering-Eldad-Eilam/dp/0764574817/)<br>
[Reverse Engineering](https://en.wikipedia.org/wiki/Reverse_engineering)<br>
[A Introduction to GCC – Brian Gough](http://www.amazon.com/An-Introduction-GCC-For-Compilers/dp/0954161793/)<br>
[Compiler, Assembler, Linker and Loader: A Brief Story](http://www.tenouk.com/ModuleW.html)<br>
[Basics of GCC compilation process](http://mylinuxbook.com/basics-of-gcc-compilation-process-explained/)<br>
[GNU C Compiler Internals/GNU C Compiler Architecture](http://en.wikibooks.org/wiki/GNU_C_Compiler_Internals/GNU_C_Compiler_Architecture)<br>
[GCC front-end Whitepaper - Andi Hellmund](http://blog.lxgcc.net/wp-content/uploads/2011/03/GCC_frontend.pdf)<br>
[Programming language](https://en.wikipedia.org/wiki/Programming_language)<br>
[List of programming languages by type](https://en.wikipedia.org/wiki/Categorical_list_of_programming_languages)<br>
[C (programming language)](https://en.wikipedia.org/wiki/C_(programming_language))<br>
[Low-level programming language](https://en.wikipedia.org/wiki/Low-level_programming_language)<br>
[Assembly Language](https://en.wikipedia.org/wiki/Assembly_language)<br>
[Machine code](https://en.wikipedia.org/wiki/Machine_code)<br>
