# A-Sin-To-Err

Cipher Machine Using [4 bit Nibbler CPU](http://bigmessofwires.com/nibbler).

![Photo of Nibbler cipher machine](nibbler.jpg?raw=true)

Encoding
--------

Before encryption, plaintext needs to be converted to a stream of decimal digits. One way 
to do it is by using [straddling checkerboard](https://en.wikipedia.org/wiki/VIC_cipher#Straddling_checkerboard).

|       | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|-------|---|---|---|---|---|---|---|---|---|---|
|       | A |   | S | I | N |   | T | O | E | R |
| **1** | B | C | D | F | G | H | J | K | L | M |
| **5** | P | Q | U | V | W | X | Y | Z | . | / |

Letters in the unmarked row are represented by one digit (column number), e.g., N becomes 4. 

Letters in the other two rows are represented by two digits (row, then column), e.g., W becomes 54.

Numbers are enclosed in / symbols (59), then each digit is repeated twice, e.g., 42 becomes 59442259.

Example encoding of text as stream of digits using straddling checkerboard:

| Plaintext | H | E | L | L | O | W | O | R | L | D |
|-----------|---|---|---|---|---|---|---|---|---|---|
| Encoded   | 15|  8| 18| 18|  7| 54|  7|  9| 18| 12|

From the command line:
    echo "Hello world" | ./straddling_checkerboard
    1581 8187 5479 1812  

Encryption
----------

Message begins with 8 digit nonce followed by stream of encrypted digits. Nonce is not
secret, but every value should only be used once. It is similar to page identifier of one 
time pad.

To encrypt, type nonce followed by encoded stream and record encrypted message shown on 
the LED display.

| Nonce     | 5551 | 8424 |      |      |      |      |
|-----------|------|------|------|------|------|------|
| Encoded   |      |      | 1581 | 8187 | 5479 | 1812 |
| Message   | 5551 | 8424 | 3279 | 9875 | 7770 | 9178 |

From the command line:

    echo "5551 8424 1581 8187 5479 1812" | ./a_sin_to_err
    5551 8424 3279 9875  7770 9178

Decryption
----------

To decrypt, type message and record decrypted stream shown on the LED display.

| Message     | 5551 | 8424 | 3279 | 9875 | 7770 | 9178 |
|-------------|------|------|------|------|------|------|
| Decrypted   |      |      | 1581 | 8187 | 5479 | 1812 |

From the command line:

    echo "5551 8424 3279 9875  7770 9178" | ./a_sin_to_err
    5551 8424 1581 8187  5479 1812

Decoding
--------

Ignore first 8 digits (nonce) and decode the rest using straddling checkerboard. Example:

| Decrypted   | 15|  8| 18| 18|  7| 54|  7|  9| 18| 12|
|-------------|---|---|---|---|---|---|---|---|---|---|
| Decoded     | H | E | L | L | O | W | O | R | L | D |

From the command line:

    echo "1581 8187  5479 1812" | ./straddling_checkerboard
    HELLOWORLD

Building
--------

Start by changing the key in a_sin_to_err.cpp. The key has to be
random, e.g., 128 coin tosses. Then build code generator:

    make a_sin_to_err
    cp asintoer.asm asintoer2.asm
    ./a_sin_to_err codegen >>asintoer2.asm 

Compile asintoer2.asm on Windows to create ROM for Nibbler:

    assembler -o asintoer2.bin asintoer2.asm
    
Compile optional encoder/decoder:
    
    make straddling_checkerboard

Implementation Details
----------------------

Encryption is [RC5](https://en.wikipedia.org/wiki/RC5) in [CTR mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_.28CTR.29).
For ease of implementation only one nibble from each block is used, and nibbles that
do not correspond to a BCD are simply discarded.

Encryption function applied to each digit is (20 - plaintext - key) % 10. It is 
[involutory](https://en.wikipedia.org/wiki/Involution_(mathematics)), meaning that 
f(f(x)) = x, so decryption function is the same as encryption: (20 - ciphertext - key).

Known Limitations
-----------------

Expanded key is stored in ROM because Nibbler's lack of indirect addressing and limited
ROM size makes it difficult to port rc5_setup. Besides, entering 39 digit key
every time is impractical.

Hardware Changes
----------------

The only hardware changes compared to the original Nibbler are input and output:

* Four seven-segment LED digits instead of LCD. HEF4543B decoders are connected to OUT0,
/LE lines to OUT1. Rightmost digit is bit 0 of OUT1.
* 12 key telephone-style keypad instead of four pushbuttons. Key switches are arranged in
a matrix. Columns are connected to IN0. Rightmost column is bit 0. Rows are connected to 
OUT0. Top row is bit 0.

Clock
-----

When no keys are pressed after device is turned on or reset, it works as a clock. Time is
displayed in 24 hour (military) standard, hours and minutes only. Time can be set using 
the keypad. Clock runs only when no or 4 digits have been entered so that it does not 
interfere with encryption.

Other keys
----------

"*" (star) key resets the device. "#" (hash) key shows debugging information (currently 
four least significant digits of CTR, hex values above 9 decoded as blanks). 
