---
layout: post
title: SLAE - 4th Assignment - Encoding and Decoding Gollums
date: '2013-04-12T23:30:00.001-03:00'
author: geyslan
tags: [slae, shellcode, linux, code, assembly, c, python, hacking, portuguese]
modified_time: '2013-08-20T15:41:02.420-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-937447178869282231
blogger_orig_url: http://www.hackingbits.com/2013/04/slae-4th-assignment-encoding-and.html
---

*/\*<br>
Este post é uma sequência. Para melhor entendimento, vejam:<br>
[SLAE - 1st Assignment - Shell Bind TCP](/slae-1st-assignment-shell-bind-tcp.html)<br>
[Hacking do Dia - Shell Bind TCP Random Port](/hacking-do-dia-shell-bind-tcp-random.html)<br>
[SLAE - 2nd Assignment - Shell Reverse TCP](/slae-2nd-assignment-shell-reverse-tcp.html)<br>
[O menor do mundo? Yeah? So Beat the Bits!](/o-menor-do-mundo-yeah-so-beat-bits.html)<br>
[SLAE - 3rd Assignment - Caçando Ovos?](/slae-3rd-assignment-cacando-ovos.html)<br>
\*/*

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/4th.assignment)*.

*Shell-Storm:<br>
[Tiny Execve Sh](http://shell-storm.org/shellcode/files/shellcode-841.php)<br>
[Insertion Decoder Shellcode](http://shell-storm.org/shellcode/files/shellcode-840.php)*

Olá pessoal!

Com este post prosseguiremos as missões do curso SLAE.

Gostaria de ressaltar que, no último
[vídeo](http://www.youtube.com/watch?v=agwqgm52p3I) do treinamento, Vivek
determinou que o aluno que reduzir os shellcodes e/ou tê-los aceitos em
repositórios receberá pontuação extra para a certificação.

Eu venho tentando, além de reduzir o tamanho, criar versões dos shellcodes com
propriedades singulares; exemplos são o Shell Bind TCP (GetPC) e o Shell Bind
TCP (com REUSE), este que acabou originando o Tiny Shell Bind TCP Random Port
(57 bytes).

Porém, nesta missão, Vivek foi claro quanto à originalidade do shellcode. Ou
seja, neste payload, o que importará realmente é o engendramento de um método de
inserção divergente do que ele demonstrou em aula.

##Insertion Decoder

Um Shellcode Insertion Decoder realiza o realinhamento de um shellcode
devidamente encodado com inserção de garbage bytes. Sua utilização é voltada
para a tentativa de bypass de ferramentas que detectam injeção de shellcode.

Na respectiva aula ele cria um decoder sequencial que substitui gargabe bytes ao
copiar o byte verdadeiro na sequencia correta. Inicialmente, pensei em apenas
remodelar o padrão de inserção da versão dele para, em vez de fazer o decoding
de apenas um garbage byte (x_x_x_), fazer de dois (x__x__x__). Mas nem cheguei a
amadurecer a ideia, desisti ao realizar que tal modificação não seria nem
original muito menos útil num caso concreto.

##Cachola para que te quero?!

Após alguns dias da resignação à ideia anterior, eis que me acendeu uma luz
acima da cabeça! Por que não criar um decoder com análise para qualquer padrão?
Deixando nas mãos do usuário a possibilidade de inserir o lixo no shellcode da
forma que melhor lhe aprouver? Isso sim seria original!

Why not?!

##Assembly e C (perfect couple)

O shellcode exigido na missão era o do execve_stack já corretamente encodado.
Como um plus, reduzi o tamanho dele engendrando o Tiny Execve Sh (21 bytes),
cujos código e shellcode podem ser visualizados no
[github](https://github.com/geyslan/SLAE/tree/master/4th.assignment).

Inseri o garbage byte em posições aleatórias dentro desse shellcode e, após um
estudo algorítmico concretizei o decoder em asm como podem ver abaixo.

{% gist geyslan/5373202 %}

{% include image.html url="http://images-onepick-opensocial.googleusercontent.com/gadgets/proxy?container=onepick&gadget=a&rewriteMime=image%2F*&url=https%3A%2F%2Fraw.github.com%2Fgeyslan%2FSLAE%2Fmaster%2F4th.assignment%2Finsertion_decoder.png" desc="Fluxograma do 4th assignment" %}

O que ele faz é percorrer a área da memória na qual o shellcode está inserido,
comparando o byte lido com o byte lixo, e reordenando-os quando resolvidas as
condições. Dúvidas? Leia o código fonte, está bem comentado.

##Pronto para publicar!

Logo após ter feito push no git, já me preparando para escrever este post,
tweetei para o Vivek!

*Wow!*

Ele não só apenas gostou do decoder, pediu também que eu fizesse um extra:

[@geyslangb Your code is really fantastic! I am really happy and proud to have
you as a student :)](https://twitter.com/SecurityTube/status/320779933465063425)

[@geyslangb If you have time maybe you could create a py/rb script to take any
shellcode as input and give out the decoder +encoded shellcode](https://twitter.com/SecurityTube/status/320889730797559809)

Well, lets go!

##An Unexpected Journey

Nunca tinha programado em Python, a não ser modificado algum pedaço de código ou
lido rapidamente. Entretanto, sempre ouvi falar que é uma ótima linguagem para
aprender, assim como para diversos outros fins.

*The campaign begins...*

**Um obstáculo** - Uso de argumentos para receber o shellcode e demais parâmetros.

Não estava conseguindo preencher a variável shellcode da forma correta...
descobri, após garimpar os [pergaminhos](http://docs.python.org/) do python, que
era por conta da necessidade do uso de encoding. Mas mesmo usando o encode(),
este baggins não conseguia fazer com que a string shellcode ficasse como
bytearray. Até que o wizard [sigsegv](http://bughunter.tecland.com.br/) empunhou
o seu cajado proclamando: error handler surrogateescape. Valeu, sig!

**Outro obstáculo** - Bytes, Strings...

Continuando na campanha, precisei comparar, em alguns casos, string a byte...
como era meu primeiro contato com a linguagem quase fui cozido por alguns trolls
até me lembrar de umas dicas dadas pelo [Pedro
Fausto](https://twitter.com/pedrofaustojr), um Dwarf que guerreia distante do
Reino perdido de Erebor. hex() e int() foram suas armas. Valeu, Pedro!

##Epílogo

{% gist geyslan/5376542 %}

Uso:

{% highlight console %}
$ ./insertion_encoder.py -h
$ ./insertion_encoder.py -g f3 -p xxbbxb -e f1f1 -s $'\x31...\x80'
{% endhighlight %}

[@geyslangb @felipensp @pedrofaustojr and the code really looked 133t on my
iPhone screen ...
;)](https://twitter.com/SecurityTube/status/321804219952791552)

*Not at all, Vivek.*

*Galadriel: Why the Halfling?*

*Gandalf: Saruman believes it is only great power that can hold evil in check,
but that is not what I have found. I found it is the small everyday deeds of
ordinary folk that keep the darkness at bay... small acts of kindness and love.
Why Bilbo Baggins? That's because I am afraid and it gives me courage.*

**\o/**
