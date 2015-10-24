---
layout: post
title: Crackme 03 - Source Code
date: '2013-05-06T20:00:00.000-03:00'
author: geyslan
categories: blog
excerpt:
tags: [linux, code, assembly, python, elf, crackme, cryptography, hacking, reverse engineering, portuguese]
share: true
modified: '2013-08-20T15:38:59.767-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-8457197090012722511
blogger_orig_url: http://www.hackingbits.com/2013/05/crackme-03-source-code.html
---

E não demorou quase nada!

<!--more-->

Parabéns ao [Fernando Mercês](https://twitter.com/MenteBinaria) e ao [Andrey
Arapov](https://twitter.com/andreyarapov) que quebraram rapidinho o binário
utilizando formas distintas e super interessantes.

Este é o walk through (análise dinâmica) do Andrey.<br>
[http://www.nixaid.com/reverse_engineering/crackme_v3_walkthrough](http://www.nixaid.com/reverse_engineering/crackme_v3_walkthrough)

Abaixo o do Fernando Mercês (análise estática).<br>
[https://groups.google.com/d/msg/brasil-underground/IByA_8XcppU/P9Oqzn9cFBwJ](https://groups.google.com/d/msg/brasil-underground/IByA_8XcppU/P9Oqzn9cFBwJ)

[* Código fonte no Github *](https://github.com/geyslan/crackmes/blob/master/src/crackme.03.asm) =D
{: style="text-align: center;"}

Resumindo, o ELF Header do crackme.03.32 foi feito com base no [Teensy ELF
Header](http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html) do [Brian
Raiter](http://www.muppetlabs.com/~breadbox/), ou seja, totalmente à mão, e não
pelos assembler e linker (este último que nem foi necessário). Por isso a
"mágica" do entrypoint estar na área que pertenceria somente ao Header. ;) Para
conseguir isso, utilizei fields do header que são desprezados na inicialização
do binário, aglutinando-os e brincando com eles para atingir o "mal necessário".

Outro aspecto do binário é o desofuscamento dinâmico da string 'Omedetou' por
meio do algoritmo uzumaki.

Há, também, dois checksums: um do header (com chave criptografada) e o outro do
binário inteiro (com chave simples).

Mas tudo isso foi implementado apenas para desviar o cracker do caminho mais
simples que é o "patch the jump".

Espero que tenham gostado. Até a próxima! o//

##Mais Informações

Crackme - Wikipedia
A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux - Brian Raiter
System V Application Binary Interface
System V ABI, Intel386 Architecture Processor Supplement
