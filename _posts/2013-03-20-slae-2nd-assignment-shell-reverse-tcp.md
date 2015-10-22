---
layout: post
title: SLAE - 2nd Assignment - Shell Reverse TCP
date: '2013-03-20T00:40:00.000-03:00'
author: geyslan
tags: [slae, shellcode, linux, code, assembly, hacking, reverse engineering, portuguese]
modified_time: '2013-06-06T22:44:21.128-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-1069667985052517024
blogger_orig_url: http://www.hackingbits.com/2013/03/slae-2nd-assignment-shell-reverse-tcp.html
---

*/\*<br>
Este post é uma sequência. Para melhor entendimento, vejam:<br>
[SLAE - 1st Assignment - Shell Bind
TCP](/slae-1st-assignment-shell-bind-tcp.html)<br>
[Hacking do Dia - Shell Bind TCP Random Port](/hacking-do-dia-shell-bind-tcp-random.html)<br>
\*/*

<!--more-->

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/2nd.assignment)*.

*\<UPDATE\><br>
O [shellcode](http://shell-storm.org/shellcode/files/shellcode-833.php) final
deste post foi aceito no repositório [Shell-Storm](http://www.shell-storm.org/).<br>
Tks Jonathan Salwan.<br>
\</UPDATE\>*

##Segundo Exame

###Criar um shellcode de Shell Reverse TCP

- Executar um shell ao conectar no host reverso.
- Tornar fácil a configuração dos IP e porta.

Como material, analisar o linux/x86/shell_reverse_tcp do Metasploit usando o libemu.

##libemu - sctest - strace - man

Comecei gerando o fluxograma do shellcode do Metasploit.

{% highlight console %}
$ msfpayload linux/x86/shell_reverse_tcp LHOST=127.0.0.1 R | /opt/libemu/bin/sctest -Ss 100000 -vvv -G shell_reverse_tcp_metasploit.dot

$ dot shell_reverse_tcp_metasploit.dot -T png -o shell_reverse_tcp_metasploit.png
{% endhighlight %}

{% include image.html url="https://raw.github.com/geyslan/SLAE/master/2nd.assignment/shell_reverse_tcp_metasploit.png" desc="Fluxograma do shell_reverse_tcp_metasploit" %}

Vê-se que, diferentemente dos shell_bind_tcp nos quais trabalhamos nos posts
anteriores, o shell_reverse_tcp, logo após a criação do socket (socket), faz a
duplicação (dup2) dos files descriptors e já efetua a conexão (connect) nos
endereço e porta definidos para finalmente executar (execve) o "/bin/sh".

Usei também o [strace](http://linux.die.net/man/1/strace) para analisar o
[netcat](http://linux.die.net/man/1/nc), como complemento ao libemu. O que me me
poupou tempo, pois não precisei reconstruir o binário para entender melhor a
syscall [connect](http://linux.die.net/man/3/connect); e nem é necessário
comentar que o uso do man deixou o meu .bash_history um pouquinho mais gordo. =D

{% highlight console %}
$ nc -l 127.0.0.1 55555
{% endhighlight %}

Em outro terminal.

{% highlight console %}

# strace -e socket,connect nc 127.0.0.1 55555
...
socket(PF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(55555), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
{% endhighlight %}

Com as novas informações anotadas e revisadas, construí de primeira o shellcode
enxuto em assembly nasm.

{% gist geyslan/5201682 %}

Por conta da possibilidade trivial de se configurar a porta e o IP, o shellcode
só estará livre de null-bytes caso esses valores não tenham numeração em hexa
x00. Entretanto, se o null-byte realmente impossibilitar a utilização do
shellcode, temos outras maneiras de darmos bypass. Em próximos posts veremos
isso.

O shellcode final teve um aumento de 4 bytes, ficando com  72 (metasploit = 68).
Esse acréscimo se deu pela propriedade da configuração do IP e da Porta nos seus
primeiros bytes. Mesmo com as instruções diferentes, o resultado foi igual.
Vejam.

{% include image.html url="https://raw.github.com/geyslan/SLAE/master/2nd.assignment/shell_reverse_tcp.png" desc="Fluxograma do shell_reverse_tcp" %}

###Testando

{% highlight console %}
$ gcc -m32 -fno-stack-protector -z execstack shellcode.c -o shellcode
{% endhighlight %}

*Terminal host*
{% highlight console %}
$ nc -l 127.1.1.1 55555
{% endhighlight %}

*Terminal cliente*
{% highlight console %}
$ ./shellcode
{% endhighlight %}

*Terminal host novamente*
{% highlight console %}
$ netstat -anp | grep shell
tcp        0      0 127.0.0.1:51600   127.1.1.1:55555   ESTABLISHED   976/shellcode
{% endhighlight %}

##Reversing (Gotcha)

Mais um shellcode construído: Shell Reverse TCP (Linux/x86) com IP e porta
facilmente configuráveis (segundo ao quinto e nono ao décimo byte,
respectivamente).

*P.S.* Se você encontrar alguma forma de reduzir a quantidade de bytes do
shellcode apresentado, entre em contato para discutirmos. Com todo prazer,
farei as alterações colocando os devidos créditos.

*[]*

##Mais Informações

[SLAE - SecurityTube Linux Assembly Expert](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)<br>
[libemu](http://libemu.carnivore.it/)<br>
[Metasploit](http://www.metasploit.com/)<br>
[Metasploit Unleashed - Offensive Security](http://www.offensive-security.com/metasploit-unleashed/Main_Page)<br>
[Berkeley Sockets](https://en.wikipedia.org/wiki/Berkeley_sockets)<br>
[Shell-Storm](http://www.shell-storm.org/)<br>
[exploit-db](http://www.exploit-db.com/)<br>
[Project Shellcode](http://www.projectshellcode.com/)<br>
[Endianness](https://en.wikipedia.org/wiki/Endianness)<br>
[Linux Assembly](http://asm.sourceforge.net/)<br>
[Introdução à Arquitetura de Computadores - bugseq team - Tiago Natel (i4k)](https://code.google.com/p/bugsec/wiki/AssemblyArqComp)<br>
[Construindo Shellcodes - Victor Mello (m0nad)](http://0fx66.com/files/zines/cogumelo-binario/edicoes/1/ConstruindoShellcodes.txt)<br>
[Understanding Intel Instruction Sizes - William Swanson](http://www.swansontec.com/sintel.html)<br>
[The Art of Picking Intel Registers - William Swanson](http://www.swansontec.com/sregisters.html)<br>
[Smashing The Stack For Fun And Profit - Aleph One](http://insecure.org/stf/smashstack.html)<br>
[Get all shellcode on binary - commandlinefu](http://www.commandlinefu.com/commands/view/6051/get-all-shellcode-on-binary-file-from-objdump)<br>
