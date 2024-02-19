# Github Signed Commit
SOURCE: https://www.youtube.com/watch?v=1vVIpIvboSg&t=0s [CREATING GPG KEY]
### CHECK IF YOU HAVE EXISTING KEYS
> gpg --list-keys

### GENERATE A NEW KEY
> gpg --full-generate-key

### OPTIONS
- (1) RSA and RSA (default)
- 4096 size
- set an expiry base on you preference, 1y is recommended
- add your info including your email to associate with the key and passphrase (don't forget it)
- generate the key

### CHECK IF A KEY WAS GENERATED
> gpg --list-keys

### UPDATE EXISTING KEY [RENEW IF IT EXPIRES]
> gpg --edit-key [email-associated]
> list
> key 0   // select the first key you want to edit
> expire  // change the key expiry date
> save    // save the updated key

THE KEY WAS UPDATED


### CHANGE PASSPHRASE
> gpg --passwd [email-associated]
- enter your old passphrase
- enter your new passphrase
- enter again your new passphrase
DONE

### EXPORTING YOUR GPG PUBLIC KEY
> gpg --export --armor [email-associated]

copy your public key

### IT SHOULD LOOK LIKE THIS
```
-----BEGIN PGP PUBLIC KEY BLOCK-----

mQINB..............................
...................................
=ng5y
-----END PGP PUBLIC KEY BLOCK-----
```
### ADDING YOUR GPG PUBLIC KEY TO YOUR GITHUB ACCOUNT
- GO TO YOUR PROFILE SETTINGS
- GO TO 'SSH and GPG keys'
- ADD A NEW GPG KEY
- Paste your COPIED GPG Key

### IN GIT CONFIG, ADD YOUR SIGNING KEY
> git config --global user.signingkey [email-associated]

DONE

SOURCE: https://www.youtube.com/watch?v=4166ExAnxmo [Signing and Verifying Git Commits on the Command Line and GitHub]

### CREATING A SIGNED COMMIT AND VERIFYING IT
> git add -A
> git commit -S -m "Some Commit" // this is a signed commit
> git log // to view the commits
> git log --show-signature // show the commit with signature

### CREATING A SIGNED TAG
> git tag -sm "Message" v1
### VERIFY A TAG
> git tag v1 -v
