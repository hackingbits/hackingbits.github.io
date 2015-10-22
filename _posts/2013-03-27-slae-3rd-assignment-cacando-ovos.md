---
layout: post
title: SLAE - 3rd Assignment - Caçando Ovos?
date: '2013-03-27T12:07:00.001-03:00'
author: geyslan
tags: [slae, shellcode, linux, code, assembly, c, hacking, portuguese]
modified_time: '2013-06-06T22:43:20.144-03:00'
thumbnail: http://1.bp.blogspot.com/-BJ2Zk5Um6mo/UVMJb5T9uaI/AAAAAAAAAIw/u2s6Z8o-6Js/s72-c/Aptitude_moo.png
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-2827188970984295225
blogger_orig_url: http://www.hackingbits.com/2013/03/slae-3rd-assignment-cacando-ovos.html
---

*/\*<br>
Este post é uma sequência. Para melhor entendimento, vejam:<br>
[SLAE - 1st Assignment - Shell Bind TCP](/slae-1st-assignment-shell-bind-tcp.html)<br>
[Hacking do Dia - Shell Bind TCP Random Port](/hacking-do-dia-shell-bind-tcp-random.html)<br>
[SLAE - 2nd Assignment - Shell Reverse TCP](/slae-2nd-assignment-shell-reverse-tcp.html)<br>
[O menor do mundo? Yeah? So Beat the Bits!](/o-menor-do-mundo-yeah-so-beat-bits.html)<br>
\*/*

<!--more-->

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/3rd.assignment)*.

##Terceira Missão (Caça aos bits)

Olá! Continuemos com a terceira missão do SLAE.

Nesta empreitada, vamos caçar ovos... não,
[cooler](https://plus.google.com/100365453402860467427), não vai ser um elefante
engolido por uma jiboia, muito menos achocolatados. :D []'s

{% include image.html url="http://1.bp.blogspot.com/-BJ2Zk5Um6mo/UVMJb5T9uaI/AAAAAAAAAIw/u2s6Z8o-6Js/s1600/Aptitude_moo.png" desc="aptitude moo" %}

{% include image.html url="http://3.bp.blogspot.com/-3xIyqUvg8sQ/UVMJaOvsHzI/AAAAAAAAAIs/27Nx8y-wh-g/s1600/_resized_elefante_copy.jpg" desc="O Pequeno Príncipe - Antoine de Saint-Exupéry" %}

###3rd Assignment

- Estudar sobre shellcodes egg hunter.
- Criar uma demonstração funcional de um.
- Essa demonstração deve ser configurável para diferentes payloads.

O resultado da pesquisa sobre egg hunting foi escasso. Encontrei apenas um
[shellcode](http://shell-storm.org/shellcode/files/shellcode-784.php) do tipo
para Linux. Fiz alguns testes mas logo vi que ele não era funcional, pois
tentava ler toda a VAS do usuário (0x0 a 0xffffffff) resultando em SIGSEGV
quando batia na porta de uma área protegida. A ideia principal que era percorrer
a memória num looping eu já tinha abstraído, entretanto, como fazer isso de
forma segura?

Perseverei no google e mudei a pesquisa de linux egg hunter para "windows" egg
hunter. Bingo! Encontrei um paper fantástico escrito por skape em 2004, [Safely
Searching Process Virtual Address
Space](http://hick.org/code/skape/papers/egghunt-shellcode.pdf). Esse artigo,
felizmente, além de abordar o windows, descreve duas formas no linux de se
verificar se o usuário (aplicativo) tem permissões de leitura a dado offset. A
primeira, a qual optei, é através da syscall
[access](http://linux.die.net/man/2/access) e a segunda é através da
[sigaction](http://linux.die.net/man/2/sigaction). Não sei por que tive que
colocar "windows" na pesquisa; o que importa é que encontrei o que precisava.

*Ok! Mas você não disse para que serve mesmo um egg hunter!*

Perdão, vamos lá! Em alguns casos de exploração encontramos pequenas janelas de
buffer disponíveis para injeção de código, nas quais um shellcode de tamanho
maior seria inútil. Nessa mesma janela poderíamos injetar um Egg Hunter,
obviamente menor que o shellcode principal, para vasculharmos a memória em busca
do nosso shellcode (egg). Em suma, o Egg Hunter é um tipo de stager shellcode.

*Mas como nosso shellcode principal (egg) estaria na memória?*

Isso vai depender da aplicação explorada, das formas de entrada de dados etc.
Como exemplos poderíamos ter um browser que carregaria o egg de um html, um mp3
player que carregaria o egg de um mp3 ou m3u, um programa genérico que leria
todo o conteúdo de um arquivo de configuração com o egg implantado.

Compreendendo as explicações do skape a respeito do seu Egg Hunter (access),
complementei-o com controle de lixo dos registers e limpando o DF ([Direction
Flag](http://en.wikipedia.org/wiki/Direction_flag)) para se garantir o
incremento de EDI no uso da [SCASD](http://faydoc.tripod.com/cpu/scasd.htm).

O resultado foi este.

{% gist geyslan/5254380 %}

Fluxograma gerado no r2 ([radare](http://www.radare.org/)).

{% include image.html url="https://raw.github.com/geyslan/SLAE/master/3rd.assignment/egg_hunter.png" desc="Fluxograma gerado no r2 (radare)" %}

Para facilitar a exemplificação, o Egg já se encontra no próprio Egg Hunter
Launcher.

{% gist geyslan/5254424 %}

A configuração da assinatura do Egg é feita mudando-se os seus oito primeiros
bytes; já no Egg Hunter, do 25° ao 28° byte. Optei por usar opcodes que não
interferem no fluxo (nops, pushs), mesmo eles sendo ignorados no último JMP.

É importante ressaltar que os quatro últimos bytes, no caso do Egg, são uma mera
repetição dos quatro primeiros; tal medida visa a fortificar a identificação
evitando-se falso-positivos e faz parte do algoritmo de identificação do Egg
Hunter, assim, se desejarem modificar os quatro primeiros bytes, repliquem a
mesma mudança nos últimos.

###Testando

{% highlight console %}
$ gcc -m32 -fno-stack-protector -z execstack egg_hunter_shellcode.c -o egg_hunter_shellcode
$ ./egg_hunter_shellcode
Egg Mark
{% endhighlight %}

##Missão Cumprida

Bom pessoal, é isso. Boa caça aos ovos!

##Mais Informações

[SLAE - SecurityTube Linux Assembly Expert](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)<br>
[Safely Searching Process Virtual Address Space - skape](http://hick.org/code/skape/papers/egghunt-shellcode.pdf)<br>
[Virtual Address Space](http://en.wikipedia.org/wiki/Virtual_address_space)<br>
[Anatomy of a program in memory - Gustavo Duarte](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory)<br>
[Cogumelo Binário](http://0fx66.com/files/zines/cogumelo-binario/)<br>
[radare](http://radare.org/)<br>
[bokken](http://inguma.eu/projects/bokken)<br>
