StripeCTF2.0
============

The exploit targets a password database which for the sake of security breaks
passwords down into multiple chunks. Each chunk is stored in separate chunk 
servers. That is, if the password is 12 characters long and there are 4 chunk
servers, chunk server 1 will contain the first 3 characters of the password,
chunk server 2 will contain the next 3 characters and so on. The main server
receives requests for validating passwords. It then validates each individual
password chunk against each corresponding chunk server. The notion behind this 
practice is that if the main server gets compromised, the password will not
be compromised. Or, if a chunk server gets compromised only part of the password
gets compromised.

To understand how the exploit works, its is important to understand how the
password server validates the passwords. So, here are the relevant details of
the password database implementation:

  1. It communicates with the chunk servers via sockets.
  2. It breaks the password down into chunks. It asks chunk server 1 to validate
     the first chunk. If the chunk is correct, it moves on to the second chunk 
     by asking chunk server 2 to validate it, and so on. However, if a 
     particular chunk server declines a particular chunk, the password database 
     stops.
  3. It implements a delay mechanism to prevent timing attacks.
  
Another important piece of the puzzle is that the password database allows to
specify a web-hook or endpoint. This web-hook is called by the password database 
once the validation of the password is completed. This exposes the port the 
operating system allocated for the connection.

Unless an operating system has been modified to be extremely secure, most 
operating systems allocate ports for outbound connections sequentially. That is, 
if the last port used was port X, the next port to be used will be X+1.

Fortunately, the password database and the chunk severs were all running on the
same machine. Hence, making it vulnerable to port counting. This vulnerability
allows the attacker to count port increments to infer which chunk server failed.
Therefore, enabling the attacker to make a biased brute-force attack on each
individual chunk rather than on the entire password.

Lets say you know what the last used port was. This is easily determined by 
sending a random request and having the server contact the web-hook. Lets call
this base port X. Next, you send your first password attempt, and the server
will break it down into chunks and contact chunk server 1, effectively 
incrementing its port count to X+1. Lets say that chunk server 1 returns a
failing answer. The database server will then contact the web-hook to notify it
that the password is incorrect. By doing so, the port count will get incremented
again by one, making the current port X+2.

The web-hook knows that the base port was X and that the port that the database
server is using to communicate with it is X+2. It also nows that one of the
increments is due to the current communication. So, the web-hook can infer that
while validating the password, the database server made (X+1) - X connections.
That is, just one connection. And this connection is likely to have been made
to chunk server 1. This inference further allows to infer that the failing
chunk was the first chunk of the password.

If the first chunk would have been correct, the database server would have then
incremented the base port by one when connecting to the first chunk server
and then again by one by connecting to the second chunk server. And, one last 
time when connecting the the web-hook. Making the exposed port number X+3.

All the exploiting code needs to do is to implement a web-hook to determine 
which chunk server failed and then brute-force that specific chunk until that 
specific chunk server stops failing. The code will brute-force each individual 
chunk until it receives a success request on the web-hook.

The exploit is still a brute-force algorithm. But since it is biased, it makes
the algorithm much more efficient and the chances of success much more likely.
Brute-forcing a 12 long password that only contains numbers could take 10^12 
tries. Brute-forcing 4 chunks of length 3 also containing only numbers would 
take at most 4*(10^3) or 4000 tries.
