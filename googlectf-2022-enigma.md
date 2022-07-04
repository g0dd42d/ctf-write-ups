##  Google CTF 2022 - ENIGMA
### Maple Mallard Magistrates (MMM)

Blechley's Brightest:  
REDACTED

<br />  
Hi all! I'm REDACTED with MMM. This is how we cracked the code and <strike>won the war</strike> got first blood!  

 <blockquote>
<b>Crypto Challenge: ENIGMA  [500 Points / 9 Solves]</b>

Cryptanalysis has advanced a long way.  
Can you break Enigma without a known plaintext?  
The flag is written in all caps, with spaces and punctuation between words. It does not contain quotes.

[Attachment](https://storage.googleapis.com/gctf-2022-attachments-project/c549d32fbc247412db5a78293bf0bee9a51101afe9b3a5f302e467e987b6e0bd784d70bde0ad323c87a12c814dd8e4402ff92574a9f5e627f9ec4f818f44e413)
</blockquote>

In the attachment, we are provided a lot of information to get started on this challenge. Most of it is provided in the challenge attachment's README.


___
### Our Enigma

The Enigma machine  was a collection of cryptographic ciphering machines developed at the end of World War I, with the later German military models, which utilized a plugboard, being the most complex. These machines underwent many improvements as they supported military operations throughout WWII. For this challenge, we are provided a directory containing C++ code that emulates the 1939 M3 model Enigma machine. The M3 supports up to 8 rotors, 5 military standard wheels plus 3 specific to the Kriegsmarine (German Navy). Each cipher wheel supports 26 ASCII characters. Our challenge code realistically uses 3 rotor slots, rotor choices I - VIII (including the 3 Naval rotors), reflector B, and 10 plugboard pairs. 

Just using these options, the number of possible combinations is as follows:
- Using any three wheels at a time theoretically gives us 336 (8x7x6) possible rotor orders.
- With 26 starting positions for each rotor, we get 17,576 (26^3) possible rotor configurations. 
- There are 26! possible letter arrangements, though a plugboard can only make 10 pairs. This gives us 20 letters pairings, so we divide by 6!. The order of the 10 pairs does not matter, so we divide by 10!. The order of the letters in the pair does not matter, so we divide by 2^10. This ultimately yields 150,738,274,937,250 plugboard combinations.  

This means that our M3 emulator could theoretically support 336 * 17576 * 150,738,274,937,250 = *890 quintillion* possible ways to configure the machine. However there are operational quirks and constraints, which are described below, that greatly reduce this number.

In addition to the Enigma emulator, the attachment also contains many other useful and necessary information, including a number of previously-decrypted sample messages. The message files contain the following:
- `.original.txt`: The original message to be encrypted.
- `.plaintext.txt`: The same message, formatted for Enigma (via `encode.py`).
- `.ciphertext.txt`: The ciphertext after being encrypted via the Enigma
  machine.
- `settings`: The settings used for that ciphertext.
- `.fernet.txt`: The original message, encoded with the Fernet cipher (via
  `encrypt_modern.py`).

All of the messages for this challenge are written in German. However, the Enigma only supports the 26 ASCII capital letters, so German operators would lossily encode messages using substitutes for characters such as ß, umlauts, punctuation, and/or spaces. This challenge employs these subsitutions, and we are provided a mapping of these characters in the source code and README. 

*As for the flag*, throughout WWII, there would often be multiple messages that discussed the same topic, and breaking any message would leak sensitive information. This remains true for this challenge. We are provided with two ciphertext messages without their corresponding plaintext - one transmission from May 12, 2022 and another from May 14. They are *not* the same message, but they both describe the same flag such that we only need to break one of them. However, unlike WWII, all messages include only the message itself encrypted with a specific key. In reality, Germany used a daily and per-message key. To translate the transmission, the Google CTF organizers were kind enough to provide a mechanism to verify that the message had been decoded correctly. Given the lossy subsitution and lack of German language skills, parsing the decrypted German communications can be difficult. Each message file contains a Fernet encryption of the plaintext which can be decrypted using the Enigma encryption parameters of the message (rotor/ring/plugboard settings). The included python script `encrypt_modern.py` can be used to decode the translated message containing the flag -- or so we thought.

Finally, we are provided a number of realistic constraints.
<blockquote>
While an Enigma machine could accept multiple copies of the same rotor, each
machine was only shipped with one copy of each. Therefore, no key includes a
duplicated rotor.

Germany generated its codebooks by rolling dice. However, it enforced rules
meant to ensure there was "enough variation" between days. If those rules
were violated, dice would be rerolled. Those rules are:
- Every day must include at least one "naval rotor" (rotors VI-VIII).
- The same naval rotor cannot be used in the same position on two consecutive
  days.

These constraints apply in this challenge. While certain other randomness
constraints were applied in some divisions during certain time periods, only the
restrictions mentioned here or enforced by `machine.cc` apply to this challenge.
Keys are otherwise random.
</blockquote>

___

### Finding the right tool for the job.

Through a bit of searching, we considered numerous ciphertext-only Enigma breaking solutions as writing the program would take time we didn't have. One very interesting project stood out as particularly promising. The **[M4 Message Breaking Project](https://www.bytereef.org/m4_project.html)** was created by Stefan Krah in 2006 to break three unbroken Kriegsmarine transmissions from November of 1942 during World War II. The encrypted transmissions were intercepted during the Triton Blackout, a 10-month span during which  Blechley was unable to decode Germany's communications after the transition to the more complex Enigma M4 TRITON key system. Due to the timing of the transmissions, the encryption mechanism could be assumed. The ciphertexts were offered by Ralph Erskine as a challenge to the <i>Journal Cryptologia</i> on [6 December, 1995](https://web.archive.org/web/20060228033013/http://members.fortunecity.com/jpeschel/erskin.htm) so "that people may try to succeed where Bletchley Park failed!" Using the computing power of thousands of participants (remember this is circa-2006), the M4 Project was ultimately successful in decrypting all 3 of these messages. A great story about the messages' origin and Captain Hartwig's Uboat can be found [here](https://ciphermachinesandcryptology.com/en/m4project.htm).

Anyway, the M4 Project's **[enigma-suite](https://www.bytereef.org/enigma-suite.html)** program implements brute-force and hill climbing or Index of Coincidence to decode messages. Since the plugboard settings occupy the majority of the keyspace, the program iterates only through the ring and rotor settings, implementing the hill climbing algorithm to optimize the plugboard settings via a scoring function. However, this process is still way too slow using the original enigma-suite tools. Luckily, due to the success of the program, many newer projects adapted its design to work much better with modern computers. To solve this challenge, we ultimately settled on Alex Shovkoplyas's **[enigma-cuda](https://github.com/VE3NEA/enigma-cuda)**, a GPU-accelerated command-line tool for ciphertext-only Enigma cryptanalysis, based on the enigma-suite program. Improving on the enigma suite, enigma-cuda employs partial exhaustion of the plugboard, starting the hillclimb with an empty plugboard rather than randomly plugging plugs from the start. Not doing so decreases the chance that those six unswapped letters are correct. The repository includes a few more key features, including support for the Enigma M3, key ranges and a <i>1941 dataset of German n-gram frequencies!</i> This allows us to easiy test our known plaintexts with their corresponding ciphertexts and quickly move on to solving the challenge.
___

### Cracking the Code

#### Testing a known message.

Using my Latptop's RTX 2070, I tested the program with the message and settings from May 9th. Using the provided Enigma settings, `VI III II NRS AO BH CU DL FM GW JZ KY PX QV XKR`, I started the machine at `B:632:AA:AAA`, using the 1941 German bigram and trigram scoring datasets. It only took a few seconds to get the decrypted message, verifying the tool works.
```
enigma-cuda.exe -M M3 C:\Users\g0dd42d\Desktop\enigma-cuda-master\data\Tri_1941_ln.txt C:\Users\g0dd42d\Desktop\enigma-cuda-master\data\Bi_1941_ln.txt C:\Users\g0dd42d\Desktop\enigma-cuda-master\enigma-cuda\bin\x64_Release\messages\may_09_2022\flag.ciphertext.txt
```

```
....
Spent: 00:00:02
Pass:  1
Score: 70760
Words: -1388
Key:   B:632:DQ:JVP
Plugs: AOBHCUDLFMGWJZKYPXQV
Text:  EHE ICH DIES chrvpkl ICH en UND FURCHTBAR en WIRT gn GEN y WELCHE UNSER WOHNORT s X mol INS EINEM INNERN BAU ey ALS ftch AUF SEINER OBERFLAECHE lecyen WUERDE y WENN IRGENDEIN beoqut ENDE r WELT KOERPER y ETWA vfk DER GROESSE UNSERES MOND esajv DIE ERDE STUERZTE yan FUEHRE d AUS SICH ZUVOR EINE ALLGEMEIN gpars TEL l

Spent: 00:00:02
Pass:  1
Score: 73868
Words: -1270
Key:   B:632:DR:JVQ
Plugs: AOBHCUDLFMGWJZKYPXQV
Text:  EHE ICH DIES chrvckl ICH en UND FURCHTBAR en WIR TUN GEN y WELCHE UNSER WOHNORT s X wol INS EINEM INNERN BAU ey ALS fuch AUF SEINER OBERFLAECHE lec DEN WUERDE y WENN IRGENDEIN beoeut ENDE r WELT KOERPER y ETWA vfn DER GROESSE UNSERES MOND esajf DIE ERDE STUERZTE yan FUEHRE d MUSS ICH ZUVOR EINE ALLGEMEIN g DAR s TEL l

Spent: 00:00:02
Pass:  1
Score: 78099
Words: -764
Key:   B:632:DS:JVR
Plugs: AOBHCUDLFMGWJZKYPXQV
Text:  EHE ICH DIE SCHRECKLICHEN UND FURCHTBAR en WIRKUNGEN y WELCHE UNSER WOHNORT sowol INS EINEM INNERN BAU ey ALS AUCH AUF SEINER OBERFLAECHE LEIDEN WUERDE y WENN IRGENDEIN BEDEUTEN DER WELT KOERPER y ETWA VON DER GROESSE UNSERES MOND es AUF DIE ERDE STUERZTE yan FUEHRE y MUSS ICH ZUVOR EINE ALLGEMEINE DAR s TEL l

00:00:37 pass 1 , trying B:634:...
``` 
We can see the optimization at work. However, these parameters do *not* match the key provided to decrypt the Fernet cipher. By default, the program excludes the turning of the left rotor, because, historically, the German military restricted the message length to less than 200 characters, thus the left rotor would rarely turn. In our case, we have many more characters, so the left rotor could affect the outcome. Unfortunaly, attempting keys with and without a left hand wheel turnover (ignoring duplicates), resulted in a similar output as before. Exhausting all other configuration options, we concluded that the variance in the design approaches of the authors of the CTF challenge and cuda-enigma program would take too much time to resolve, so we moved on.  

Throwing the unparsed result in Google Translate yields the following:  
`BEFORE I SUFFER THE TERRIBLE AND TERRIBLE EFFECTS Y WHICH OUR PLACE OF RESIDENCE SOWOL IN AN INNER BUILDING EY AS WELL AS ON ITS SURFACE WOULD SUFFER Y IF ANY MEANING OF THE WORLD BODY Y ABOUT FROM THE SIZE OF OUR MOON IT TO THE EARTH`

#### Breaking the flag message.

The challenge ciphertext from May 12 contains 2641 characters - about ~1000 more than May 14, so we started there. Using the same process, I ran the program against the flag message ciphertext. After about an hour, I had text that looked like jarbled German.
```
....
Spent: 01:16:50
Pass:  1
Score: 89560
Words: -1440
Key:   B:741:SN:BJR
Plugs: AHBOCLDUFWGMJYKZPVQX
Text:  LIEBE WETTBEWERB ery DIE f LAG ge BEGINNT MIT EINEM GROSSEN jcjy EINEM GROSSEN jtj UND jfj X DARAUF FOLGT EINE OFFENE geschwiffeneklammer X DER LETZTE BUCHSTABE IST EINE GESCHLOSSENE geschwiffeneklammer X ZWISCHEN DIESEN klamm ERNST EHE n EINIGE ANDERE BUCHSTABEN X DER ERSTE BUCHSTABE ZWISCHEN DEN klam

DONE: 1 passes in 05:19:17
```

Throwing the unparsed result in Google Translate yields the following jumble:  
`DEAR COMPETITION ERY THE F LAG GE BEGINS WITH A BIG JCJY A BIG JTJ AND JFJ X FOLLOWED BY AN OPEN POLISHED CLIP X THE LAST LETTER IS A CLOSED POLISHED CLAMP X BETWEEN THESE CLAMM SERIOUS MARRIAGES N SOME OTHER LETTERS X THE FIRST LETTER BETWEEN THE KLAM`  

___ 

### Okay... so who here speaks German?

*Ideally*, we would be able to input the Enigma settings used to break the code into encrypt_modern.py to obtain the correctly parsed German message. However, the Fernet cipher key requires an exact match. All 10! permutations of the correct plugboard could be used to decrypt the enigma cipher, so it's a matter of whether or not they are sorted.
```
./encrypt_modern.py [encrypt|decrypt] {key pieces} < input > output
```
Awhile after we solved this challenge, the organizers released an improved algorithm for Fernet encryption. For example, listing plugboard pairs in a different order will not affect the validity of the key. The Fernet messages were regenerated. The first flag key was supposedly sorted, while the second flag key was not. We also learned that a nonzero ring setting was used for the first rotor. While it doesn't matter for the enigma cipher, it does for the Fernet cipher. 

No matter, however - things had seemed a bit too straightforward anyway. We got to parsing the German by hand. *=)* came in with the carry, parsing out German words as we made substitutions. Since we know the flag format - "The flag is written in all caps, with spaces and punctuation between words. It does not contain quotes." - we should be able to get fairly close to the flag. The provided `encode.py`, contained the encoding subsitutions that were applied before the message was encrypted by ENIGMA.

```
# Letter representation
text = text.replace('Ä', 'AE')
text = text.replace('Ö', 'OE')
text = text.replace('Ü', 'UE')
text = text.replace('ẞ', 'SS')
text = text.replace('ß', 'SS')

# Punctuation representation
text = text.replace('.', 'X')
text = text.replace('!', 'X')
text = text.replace("?", 'UD')

text = text.replace(':', 'XX')
text = text.replace(',', 'Y')

text = text.replace('--', 'YY')
text = text.replace('-', 'YY')
text = text.replace('/', 'YY')
text = text.replace('\\', 'YY')

text = text.replace('"', 'J')
text = text.replace("'", 'J')
text = text.replace("„", 'J')
text = text.replace("“", 'J')
text = text.replace("»", 'J')
text = text.replace("«", 'J')

text = text.replace("(", 'KK')
text = text.replace(")", 'KK')
text = text.replace("[", 'KK')
text = text.replace("]", 'KK')
text = text.replace("{", 'KK')
text = text.replace("}", 'KK')

# Strip anything that's left
text = "".join([c if (ord(c) >= ord('A') and ord(c) <= ord('Z')) else "" for c in text])
```

We made multiple passes. This process took less than an hour as we only had to parse out the section of the message that described the flag. Our notes result looked something like this:

```
DER ERSTE BTCHSTABE ZWISCHEN DEN KLAMMERN IST "D".

DER ZWEITE BUCHSTABE LWISCHEN DEN KLAMMERN IST "U".

YER DRITTE BUCHSTABE ZWISCHEN EEN KLAMMERN IST "H".

DER VIERTR BUCHSTABE ZWISCHEN DEN KLAMMMRN IST "A".

DER FÜNFTE BUCHSTWBE ZWISCHEN DEN KLAMMERN IST "S".

DARAUF FOLGT "T GE".

DANACH KOMMT "TAN".

WEITERHIN KOMMT NMN "WAS" WORAUF "TURING" FOLGT.

DIE LETZTEN BUCHSTABEN SIND "N", "I", "CHT KONNTE"

DU HAST GETAN, WAS TURING NICHT KONNT
```

The flag, just as those challenges proposed to the <i>Journal Cryptologia</i>, read "HAST GETAN, WAS TURING NICHT KONNTE", or YOU DID WHAT TURING COULDN'T. :)

Flag: CTF{DU HAST GETAN, WAS TURING NICHT KONNTE}  


#### Conclusion
We submitted the flag for this challenge around noon on July 2nd. The next solve came later that night an hour or so after the CTF organizers corrected the Fernet encryption algorithm (it's not a German language CTF after all). I think this challenge was particularly difficult as many teams probably chose to design their own code breakers. While we explored existing solvers, there were many projects that were either nonfunctional or did not support our particular implementation. The Enigma has a lot of quirks of its own, and there are many ways to go about solving it. Depending on the implementation and scoring systems, it can be difficult to get the exact parameters that were needed to decrypt the Fernet cipher with the parsed German text. The M4 project and enigma-cuda repository can also be difficult to find if you don't know to look for them, though I'm sure that there are other similar projects out there that work quite well. The history surrounding these project was fascinating though difficult to trace as they are from a much earlier internet with mostly broken reference links (wayback ftw). 


I found it most interesting that even as late as a few years ago, there are still purportedly unbroken German Enigma transmissions from WWII. Upon further reading, I found a 2005 article, <i>Breaking German Army Ciphers</i>, in the <i>Journal Cryptologia</i>, which described serveral hundred authentic German Enigma transmissions and the ciphertext-only cryptanalysis process and techniques used to break them. A few particularly short messages remained unbroken.

Thanks for reading!

#### Resources:
https://www.tandfonline.com/doi/full/10.1080/01611194.2016.1238423?needAccess=true
https://www.nsa.gov/portals/75/documents/about/cryptologic-heritage/historical-figures-publications/publications/wwii/CryptoMathEnigma_Miller.pdf?ver=2019-07-31-064622-023  
https://en.wikipedia.org/wiki/Enigma_machine
