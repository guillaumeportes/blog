# A Common Lisp Ethereum Library

In 2022 we pivoted to Web3, with the aim of figuring out concrete benefits for running games fully on the blockchain, as well as raising funds (we failed with both, unfortunately).

At the same time I had become convinced that using Common Lisp *somehow* would be beneficial to Tinka, as well as myself, and we decided to go for it and develop our software in it.

## Short(ish) summary

A lot of content (especially code) follows, and I thought it might be a good idea to include a short summary:

* This post describes the details of a Common Lisp library for interacting with the Ethereum blockchain, `eth-cl`
* `eth-cl` was written in the context of a game project (interactive story to be precise)
* The goal was to talk to Ethereum at the lowest level, through `json-rpc`
* The library includes the following facilities:
  * Account creation
  * Contract deployment
  * Contract interaction
  * Underlying transaction work
  
This was my first professional Common Lisp project, and by no means is it meant to be a reference for anything else but the learning I got from it.

## Do it yourself

When tackling a new problem (in games or otherwise), it is often very tempting to use third party software. In many cases it makes perfect sense - why reinvent the wheel?

Sometimes it is useful to do it yourself though.

After a week or so of reading up on Web3 (and Ethereum in particular), we decided that the best thing to do was to try and make a little game running on chain (this might actually have been part of a tutorial on [ethereum.org](https://ethereum.org)). There was a lot to unpack technically, and there is often nothing like producing working software in order to learn.

I wrote a very simple PvP Tic-Tac-Toe game and deployed it to a testnet. It was great to see something working, and to get a feel for the pipeline of deploying something on chain.

There was one problem though: because so much was abstracted by the libraries I used ([web3.py](https://web3py.readthedocs.io/en/stable/), in addition to quite a few frameworks of various kinds), I actually understood very little about how what I had written worked under the hood. For instance, I found it very hard to *truly* understand the relationship between gas consumption and transactions.

As I was convinced that we needed to really build a strong understanding of how Ethereum worked, I decided to write a library to interact with the chain. Of course, this would be done using Common Lisp.

## Requirements

While what we were building was not very settled to start with (we tentatively started with the idea of building a chess game), we eventually settled on making a platform to publish "Choose your own Adventure" books like [this one](https://en.wikipedia.org/wiki/Flight_from_the_Dark).

The idea was to provide authors the tools and pipelines to create those interactive stories and self-publish them on the blockchain, and to provide end users with clients to browse, purchase and "play" those books. Extra content (such as images) was to be uploaded on IPFS, for a truly decentralised platform.

The requirements for the library were relatively simple (and obvious, as that's kind of "it" in terms of what interacting with Ethereum means, since the real stuff happens in the smart contracts):

* Create accounts (we wanted to make free-to-play games, so needed "temporary" Ethereum addresses)
* Create contracts from compiled Solidity code
* Call functions on those contracts, and decode responses
* Deploy contracts to Ethereum

While we tried to do as much as possible from scratch, a couple of things were a bit too much:

* [Encoding and decoding parameters](https://docs.soliditylang.org/en/v0.8.20/abi-spec.html). This was simply something I did not take the time to do, as there was a lot involved
* Signing transactions. The cryptography library we used, [Ironclad](https://github.com/sharplispers/ironclad) did not expose everything we needed, and I struggled to make the changes for it to do so

Initially we achieved the above using Geth on a local private chain, but I could not get it to sign transactions for other chains.
I ended up using a stripped down version of web3.py (`eth_keys.backends.native.ecdsa` for signing raw transactions, and `eth_abi` for encoding and decoding parameters), and talking to it through a microservice (of course the proper way would have been to somehow have Common Lisp talk to Python, but this was much simpler as we were only using the library in the context of Docker containers deployed on AWS).

## Account

An account is a private secp256k1 key (the appropriate litterature is [here](https://github.com/ethereumbook/ethereumbook/blob/develop/04keys-addresses.asciidoc)) from which we can derive a public key and an address.

Using Ironclad, generating a new private key was simply a case of calling `(ironclad:make-private-key :secp256k1 :x (ironclad:random-data 32))`.

The fun bit was to generate the Ethereum address from it. Yes, it's "just" a hash of a subsequence of the private key, but there is added fun: each character in the address is capitalized as specified [here](https://github.com/Ethereum/EIPs/blob/master/EIPS/eip-55.md).

Here is the full code:

```
(defun compute-address (pk)
  "Computes an Ethereum address from a private key (https://github.com/ethereumbook/ethereumbook/blob/develop/04keys-addresses.asciidoc).
Capitalization is implemented using the method described here: https://github.com/Ethereum/EIPs/blob/master/EIPS/eip-55.md"
  (let* ((address (subseq (ironclad:byte-array-to-hex-string
                           (ironclad:digest-sequence :keccak/256 (subseq (getf (ironclad:destructure-private-key pk) :y) 1))) 24))
         (hash (ironclad:byte-array-to-hex-string
                (ironclad:digest-sequence :keccak/256 (ironclad:ascii-string-to-byte-array address)))))
    (loop for i from 0 to (1- (length address))
          do (when (char>= (elt hash i) #\8) (setf (elt address i) (char-upcase (elt address i)))))
    address))

(defun make-account (&optional secret)
  "Make an account from a secret string / private key."
  (let* ((bytes (if secret (ironclad:hex-string-to-byte-array secret) (ironclad:random-data 32)))
         (private-key (ironclad:make-private-key :secp256k1 :x bytes))
         (public-key (ironclad:make-public-key :secp256k1 :y (getf (ironclad:destructure-private-key private-key) :y)))
         (address (compute-address private-key)))
    `((private-key . ,private-key)
      (public-key . ,public-key)
      (address . ,address))))
```

We then can use the following three simple functions to access the data:

```
(defun address (account)
  "Returns the address of that account with leading 0x."
  (concatenate 'string "0x" (cdr (assoc 'address account))))

(defun private-key (account)
  "Returns the private key of the account with leading 0x."
  (ironclad:byte-array-to-hex-string (getf (ironclad:destructure-private-key (cdr (assoc 'private-key account))) :x)))

(defun public-key (account)
  "Returns the public key of the account."
  (cdr (assoc 'public-key account)))
```

## RPC

As we wanted to talk directly to the chain, we were going to do it through the [json-rpc](https://ethereum.org/en/developers/docs/apis/json-rpc/) interface.
Thanks to Common Lisp macros I could easily generate the functions calling the various rpc ones (using [Dexador](https://github.com/fukamachi/dexador) for sending the http requests, and [jzon](https://github.com/Zulu-Inuoe/jzon) to encode and decode to JSON).

```
(defmacro defrpc (method &optional (fn '#'identity))
  "Defines a function for a particular RPC method.
The name is converted to Common-Lisp style, i.e. eth_getBalance -> eth/get-balance.
Returns the \"result\" entry of the json output.
An optional function transforms the output, i.e. (eth/get-balance \"0xfde5e13d440a0c73b231766c76ac3268b52db346\" \"latest\") returns 10 instead of \"0xa\"."
  (labels ((camel-case-to-kebab-case (s)
             (labels ((rec (l)
                        (cond ((null l) nil)
                              ((equal (car l) #\_) (cons #\/ (rec (rest l))))
                              ((upper-case-p (car l)) (append (list #\- (char-downcase (car l))) (rec (rest l))))
                              (t (cons (car l) (rec (rest l)))))))
               (map 'string #'identity (rec (map 'list #'identity s))))))
    (let ((name (camel-case-to-kebab-case method)))
      `(defun ,(read-from-string name) (&rest params)
         (let ((response (jzon:parse (apply #'rpc (append (list *host* ,method) params)))))
           (multiple-value-bind (v p) (gethash "error" response)
             (if p
                 (error 'rpc-error :method ,name :code (gethash "code" v) :text (gethash "message" v) :data (gethash "data" v))
                 (funcall ,fn (gethash "result" response)))))))))
```

I was then able to define RPC functions like follows:

```
(defrpc "eth_getBalance" #'hex-string-to-integer)
(defrpc "eth_getTransactionCount" #'hex-string-to-integer)
(defrpc "eth_sendRawTransaction")
(defrpc "eth_getTransactionReceipt")
(defrpc "eth_call")
```

For instance, `eth_getBalance` expands into:

```
(defun eth/get-balance (&rest params)
  (let ((response
         (com.inuoe.jzon:parse
          (apply #'rpc (append (list *host* "eth_getBalance") params)))))
    (multiple-value-bind (v p)
        (gethash "error" response)
      (if p
          (error 'rpc-error :method "eth/get-balance" :code (gethash "code" v)
                 :text (gethash "message" v) :data (gethash "data" v))
          (funcall #'hex-string-to-integer (gethash "result" response))))))
```

This was the bottom layer of the library (there actually is an existing library, [web3.lisp](https://github.com/imnisen/web3.lisp), achieving the same thing).

## Transactions

Going deeper into transactions was useful for three things:

* It allowed me to look at the proper definition of a transaction, and its format (as well as understanding the different versions)
* I was able to understand gas much better, instead of trying to send various values until it works
* It exposed me to how they look in their raw form, which is useful as one often deals with long hex strings when playing with Ethereum. Being able to tell what they represent is very useful

The first step was to define the data structure that would hold the transaction objects. As often, I started with a property list, and eventually moved to a class. The format is based on the definition specified [here](https://eips.ethereum.org/EIPS/eip-1559), i.e. a EIP-1559 transaction.

```
(defclass transaction ()
  ((type 
    :initarg :type
    :initform 2)
   (chain-id
    :initform (eth/chain-id)
    :reader transaction-chain-id)
   (nonce
    :initarg :nonce)
   (max-priority-fee-per-gas
    :initarg :max-priority-fee-per-gas
    :initform (round (* 2.5 (expt 10 9))))
   (max-fee-per-gas
    :initarg :max-fee-per-gas)
   (gas
    :initarg :gas)
   (to
    :initarg :to)
   (from
    :initarg :from
    :initform (error "Must provide an account to send from.")
    :reader transaction-from)
   (value
    :initarg :value
    :initform 0)
   (data
    :initarg :data
    :reader transaction-data)
   (hash
    :reader transaction-hash)
   (receipt
    :reader transaction-receipt
    :initform nil)))
```

As you can see, this is simply exposing the fields from that definition. When creating a new transaction instance, we initialize the `max-fee-per-gas` from chain data:

```
(defmethod initialize-instance :after ((tx transaction) &key)
  ;; Initialise max-fee-per-gas
  (unless (slot-boundp tx 'max-fee-per-gas)
    (with-slots (max-fee-per-gas max-priority-fee-per-gas) tx
      (setf max-fee-per-gas (+ max-priority-fee-per-gas (* 2 (eth/gas-price)))))))
```

We then have the function translating that data into something the RPC functions will be able to consume.

```
(defun transaction-payload (tx)
  "Return the payload for tx."
  (with-slots (type chain-id nonce max-priority-fee-per-gas max-fee-per-gas gas to from value data) tx
      (let* ((result `(("type" . ,(format nil "0x~x" type))
                       ("chainId" . ,(format nil "0x~x" chain-id))
                       ("maxPriorityFeePerGas" . ,(format nil "0x~x" max-priority-fee-per-gas))
                       ("maxFeePerGas" . ,(format nil "0x~x" max-fee-per-gas))
                       ("from" . ,(address from))
                       ("value" . ,(format nil "0x~x" value)))))
        ;("nonce" . ,(format nil "0x~x" nonce))
        (when (slot-boundp tx 'nonce) (setf result (append result (list (cons "nonce" (format nil "0x~x" nonce))))))
        (when (slot-boundp tx 'data) (setf result (append result (list (cons "input" data)))))
        (when (slot-boundp tx 'to) (setf result (append result (list (cons "to" to)))))
        (when (slot-boundp tx 'gas) (setf result (append result (list (cons "gas" (format nil "0x~x" gas))))))
        result)))
```

Sending a transaction to the chain was done in two stages:

* Firstly, set the nonce and estimate gas cost (`eth/get-transaction-count` and `eth/estimate-gas` on the transaction payload)
* Secondly, encode and send the transaction

Encoding a transaction meant producing the raw transaction as described [here](https://eips.ethereum.org/EIPS/eip-1559) and signing it. As I mentioned above, we delegated signing to web3.py, but we managed to do the rest of the process.

The first thing to do was to flatten the transaction, basically by concatenating all its data into one hexadecimal string. This is what `transaction->flat` does.

```
(defun transaction->flat (tx)
  "Flattens a transaction in the format described here: https://eips.ethereum.org/EIPS/eip-1559."
    (labels ((field (x) (cdr (assoc x tx :test #'equal)))
             (from-integer (x) (ironclad:integer-to-octets (hex-string-to-integer (field x))))
             (from-string (x) (let* ((s (or (field x) ""))
                                     (s (if (zerop (length s)) s (subseq s 2))))
                                (ironclad:hex-string-to-byte-array s))))
      (list (from-integer "chainId")
            (from-integer "nonce")
            (from-integer "maxPriorityFeePerGas")
            (from-integer "maxFeePerGas")
            (from-integer "gas")
            (from-string "to")
            (from-integer "value")
            (from-string "input")
            nil)))
```

Then, that flattened transaction needs to be [RLP encoded](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/). The code seems a bit cryptic (no pun intended), but is actually quite straightforward. It is more a matter of following the documentation than anything else and encoding the parameters in the right format.

```
(defun rlp-encode (x)
  "RLP encodes the list passed following the format described here: https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/.
x is a list of byte arrays."
  (cond ((null x) (list #xc0))
        ((listp x) (let* ((encoded-x (mapcar #'rlp-encode x))
                          (lengths (mapcar #'length encoded-x))
                          (total-length (reduce #'+ lengths))
                          (total-length-bytes (map 'list #'identity (ironclad:integer-to-octets total-length)))
                          (bytes (reduce #'append encoded-x)))
                     (if (<= 0 total-length 55)
                         (cons (+ #xc0 total-length) bytes)
                         (cons (+ #xf7 (length total-length-bytes)) (append total-length-bytes bytes)))))
        (t (let*  ((bytes (map 'list #'identity x))
                   (bytes-length (length x))
                   (bytes-length-bytes (map 'list #'identity (ironclad:integer-to-octets bytes-length))))
             (cond ((and (= bytes-length 1) (<= (nth 0 bytes) #x7f)) bytes)
                   ((<= 0 bytes-length 55) (cons (+ #x80 bytes-length) bytes))
                   (t (cons (+ #xb7 (length bytes-length-bytes)) (append bytes-length-bytes bytes))))))))
```

The above function essentially recurses as long as it is being passed a list. When dealing with atoms, it adds the right data depending on the byte length of those atoms.

Getting the hash of a transaction is then simple:

```
(defun transaction->hash (tx)
  "Computes the hash of a transaction."
  (ironclad:byte-array-to-hex-string
   (ironclad:digest-sequence
    :keccak/256
    (coerce (cons 2 (rlp-encode (transaction->flat tx))) '(simple-array (unsigned-byte 8) (*))))))
```

Finally, the `encode-transaction` function:

```
(defun encode-transaction (tx pk)
  "Produces a raw transaction to be sent to the blockchain. It follows the format described here: https://eips.ethereum.org/EIPS/eip-1559.
Signing is done through a Python library as ironclad doesn't return `v`. We initially signed through geth, but signing for a different chain id seems unsupported. Sending the whole transaction structure to web3.py is not satisfactory either as the library tampers with the contents (for instance, adds a gas field to a EIP-1559 transaction.)"
  (let* ((flat (transaction->flat tx))
         (hash (transaction->hash tx))
         (signature (jzon:parse (ecdsa-raw-sign hash pk)))
         (v (gethash "v" signature))
         (r (gethash "r" signature))
         (s (gethash "s" signature))
         (concatenated (append flat (list (ironclad:integer-to-octets v) (ironclad:integer-to-octets r) (ironclad:integer-to-octets s)))))
    (concatenate 'string "0x02"
                 (ironclad:byte-array-to-hex-string
                  (coerce (rlp-encode concatenated) '(simple-array (unsigned-byte 8) (*)))))))
```

Once we have encoded our transaction, we can send it using `eth/send-raw-transaction`. This is abstracted in the following generic function and methods:

```
(defgeneric send (transaction)
  (:documentation "Send the transaction to the blockchain."))

(defmethod send :before ((tx transaction))
  (unless (slot-boundp tx 'nonce)
    (with-slots (from nonce) tx
      (setf nonce (eth/get-transaction-count (address from) "pending"))))
  (unless (slot-boundp tx 'gas)
    (with-slots (gas) tx
      (setf gas (eth/estimate-gas (transaction-payload tx))))))

(defmethod send ((tx transaction))
  "Send the transaction to the blockchain."
  (let* ((payload (transaction-payload tx))
         (encoded (encode-transaction payload (private-key (transaction-from tx))))
         (hash (eth/send-raw-transaction encoded)))
    (setf (slot-value tx 'hash) hash))
  tx)
```

One reason for using methods was to be able to specialise the behaviour for certain types of transaction: for instance, we needed to support what we called "concurrent" transactions and ensure the nonce would always be unique, which looked like this:

```
(defmethod send ((tx concurrent-transaction))
  "Lock the nonce."
  (log:info "Locking!")
  (with-redis-lock
    (with-slots (from nonce) tx
      (setf nonce (eth/get-transaction-count (address from) "pending")))
    (call-next-method)))
```

We also wanted to be able to block on transactions being mined.

```
(defgeneric wait (transaction)
  (:documentation "Wait for the transaction to be minted."))

;;; Signalled when waiting on an unsent transaction.
(define-condition unsent (error)
  ((transaction :initarg :transaction)))

(defmethod wait ((tx transaction))
  "Wait for the transaction to be minted."
  (unless (slot-boundp tx 'hash)
    (error 'unsent :transaction tx))
  (setf (slot-value tx 'receipt)
        (do ((receipt nil (eth/get-transaction-receipt (transaction-hash tx))))
            ((and receipt (not (equal 'null receipt))) receipt)
          (sleep 0.1)))
  tx)
```

I of course also wrote the equivalent decoding functions:

```
(defun rlp-decode (x)
  "RLP decodes the list passed following the format described here: https://eth.wiki/en/fundamentals/rlp.
x is a byte array."
  (cond ((zerop (length x)) nil)
        ((<= 0 (elt x 0) #x7f) (cons (ironclad:byte-array-to-hex-string (subseq x 0 1)) (rlp-decode (subseq x 1))))
        ((<= #x80 (elt x 0) #xb7)
         (let ((length (- (elt x 0) #x80)))
           (cons (ironclad:byte-array-to-hex-string (subseq x 1 (+ 1 length))) (rlp-decode (subseq x (+ 1 length))))))
        ((<= #xb8 (elt x 0) #xbf)
         (let* ((length-length (- (elt x 0) #xb7))
                (length (ironclad:octets-to-integer (subseq x 1 (+ 1 length-length)))))
           (cons (ironclad:byte-array-to-hex-string (subseq x (+ 1 length-length) (+ 1 length-length length))) (rlp-decode (subseq x (+ 1 length-length length))))))
        ((= #xc0 (elt x 0)) (append (list nil) (rlp-decode (subseq x 1))))
        ((< #xc0 (elt x 0) #xf7)
         (let ((length (- (elt x 0) #xc0)))
           (append (rlp-decode (subseq x 1 (+ 1 length))) (rlp-decode (subseq x (+ 1 length))))))
        ((<= #xf8 (elt x 0) #xff)
         (let* ((length-length (- (elt x 0) #xf7))
                (length (ironclad:octets-to-integer (subseq x 1 (+ 1 length-length)))))
           (append (rlp-decode (subseq x (+ 1 length-length) (+ 1 length-length length))) (rlp-decode (subseq x (+ 1 length-length length))))))))

(defun decode-transaction (raw)
  "Decodes a raw (hex string) transaction."
  (let ((x (rlp-decode (ironclad:hex-string-to-byte-array (subseq raw 4)))))
    (values (make-instance 'transaction :type (hex-string-to-integer (subseq raw 0 4))
                                        :chain-id (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 0 x)))
                                        :nonce (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 1 x)))
                                        :max-priority-fee-per-gas (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 2 x)))
                                        :max-fee-per-gas (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 3 x)))
                                        :gas (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 4 x)))
                                        :to (if (zerop (length (nth 5 x))) "" (format nil "0x~a" (nth 5 x)))
                                        :value (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 6 x)))
                                        :data (if (zerop (length (nth 7 x))) "" (format nil "0x~a" (nth 7 x)))
                                        :from (gethash "result" (jzon:parse (recover-address raw))))
            (ironclad:octets-to-integer (ironclad:hex-string-to-byte-array (nth 9 x)))
            (format nil "0x~a" (nth 10 x))
            (format nil "0x~a" (nth 11 x)))))
```

## Contracts

Finally, at the highest level we have contracts. We needed to be able to achieve the following:

* Load smart contract data from compiled Solidity files
* Deploy contracts to the chain
* Call functions on contracts

First, we wrap contract data into a class definition:

```
(defclass contract ()
  ((name 
    :reader contract-name)
   (data
    :reader contract-data)
   (address
    :reader contract-address)
   (call-transaction-type 
    :initform 'call-transaction
    :reader call-transaction-type)
   (deploy-transaction-type
    :initform 'deploy-transaction
    :reader deploy-transaction-type)))

(defmethod initialize-instance :after ((contract contract) &key path)
  "Build a contract object from a file."
  (log:info path)
  (when path
    (let* ((abbreviated-file (format nil "~a.~a" (pathname-name path) (pathname-type path)))
           (class-name (pathname-name path))
           (data (make-contract
                  (format nil "~a:~a" abbreviated-file class-name)
                  (format nil "~a~a.json" (make-pathname :directory (pathname-directory path)) (pathname-name path)))))
      (setf (slot-value contract 'name) class-name)
      (setf (slot-value contract 'data) data))))
      
(defun make-contract (name file)
  "Makes a contract object from a solc compiled json file.
solc --combined-json bin,bin-runtime,abi,storage-layout,hashes filename.sol -o . --overwrite"
  (gethash name (gethash "contracts" (with-open-file (in file) (jzon:parse in)))))
```

Deploying a contract is of course done through a transaction. In order to streamline things, we define a `deploy-transaction` class which fills the data from the compiled contract when instantiated.

```
(defclass deploy-transaction (transaction)
  ((contract
    :initform (error "Must provide contract.")
    :initarg :contract
    :reader transaction-contract)
   logs))
   
(define-condition wrong-number-of-parameters (error)
  ((contract :initarg :contract)
   (required :initarg :required)
   (provided :initarg :provided)))

(defmethod initialize-instance :after ((tx deploy-transaction) &key parameters)
  "Build the contract deploy transaction."
  (let* ((data (contract-data (transaction-contract tx)))
         (bytecode (bytecode data))
         (required (parameters data "constructor"))
         (count (parameters-count required))
         (encoded (if (constructor data)
                      (if (= count (length parameters))
                          (gethash "result" (jzon:parse (apply #'encode-parameters (parameters data "constructor") parameters)))
                          (error 'wrong-number-of-parameters :contract (transaction-contract tx) :required required :provided parameters))
                      "")))
    (setf (slot-value tx 'data) (format nil "0x~a~a" bytecode encoded))))
    
(defmethod wait :after ((tx deploy-transaction))
  "Set the contract address."
  (setf (slot-value (transaction-contract tx) 'address) (gethash "contractAddress" (transaction-receipt tx))))    

(defun bytecode (contract)
  "Returns the contract's bytecode."
  (gethash "bin" contract))
  
(defun parameters (contract signature)
  "Returns an entry's parameters, i.e. a signature without the name."
  (if (equal signature "constructor")
      (if (constructor contract)
          (arguments (gethash "inputs" (constructor contract)))
          "()")
      (subseq signature (position #\( signature))))

(defun parameters-count (parameters)
  "Returns the amount of parameters."
  (if (string= "()" parameters)
      0
      (1+ (count #\, parameters))))

(defun constructor (contract)
  "Returns the contract's constructor."
  (find-if #'(lambda (x) (equal (gethash "type" x) "constructor")) (abi contract)))

(defun arguments (v)
  "Returns the arguments list from an input or output list. Flattens structs."
  (labels ((rec (l)
             (cond ((null l) "")
                   ((or (equal "tuple" (gethash "type" (car l))) (equal "tuple[]" (gethash "type" (car l))))
                    (format nil
                            "(~a)~a~a~a"
                            (rec (map 'list #'identity (gethash "components" (car l))))
                            (if (equal "tuple[]" (gethash "type" (car l))) "[]" "")
                            (if (null (cdr l)) "" ",")
                            (rec (cdr l))))
                   (t (format nil "~a~a~a" (gethash "type" (car l)) (if (null (cdr l)) "" ",") (rec (cdr l)))))))
    (format nil "(~a)" (rec (map 'list #'identity v)))))
```

In order to create the `deploy-transaction`, we need to concatenate the contract's bytecode to the encoded parameters of its constructor, if there is one (encoding is done through the tiny web3.py subset I have mentioned above). The most interesting bit is the `arguments` function, which deals with structs and flattens them.

Calling a function on a contract abstracts two separate concepts:

* Calling a `pure` or `view` function results in an `eth/call` call
* Calling another type of function results in a new transaction being sent to the chain for mining

```
(defclass call-transaction (transaction)
  ((contract :initform (error "Must provide contract.")
             :initarg :contract
             :reader transaction-contract)
   (signature :initform (error "Must provide signature.")
              :initarg :signature)
   (result :reader transaction-result)
   logs))

(define-condition no-code (error)
  ((address :initarg :address)))

(defmethod initialize-instance :after ((tx call-transaction) &key parameters)
  "Build the contract call transaction."
  (with-slots (signature to) tx
    (let ((address (if (slot-boundp tx 'to) to (contract-address (transaction-contract tx))))
          (data (contract-data (transaction-contract tx))))
      (abi-entry data signature)   ; check that the signature is valid
      (unless address (error "null address"))
      (let ((code (eth/get-code address "latest")))
        (when (string= code "0x")
          (error 'no-code :address address)))
      (if (= (parameters-count (parameters data signature)) (length parameters))
          (let ((encoded (gethash "result" (jzon:parse (apply #'encode-parameters (parameters data signature) parameters)))))
            (setf (slot-value tx 'to) address)
            (setf (slot-value tx 'data) (format nil "0x~a~a" (hash data signature) encoded)))
          (error 'wrong-number-of-parameters :contract (transaction-contract tx) :required (parameters data signature) :provided parameters)))))
```

A `call-transaction` is a wrapper for a transaction that is going to call a function. This is confusing as it is used for both read only calls (i.e. `pure` or `view` functions) as well as others.
Creating a call transaction builds the right payload depending on the parameters that need to be passed to the contract function.

```
(defun call ((contract contract) &key address account signature (value 0) parameters)
  "Build a transaction, send it to eth/call and return it."
  (handler-case
      (let* ((tx (make-instance (call-transaction-type contract)
                                :contract contract
                                :signature signature
                                :value value
                                :to (if address address (contract-address contract))
                                :from account
                                :parameters parameters))
             (result (eth/call (transaction-payload tx) "latest")))
        (setf (slot-value tx 'result) (decode-result (contract-data contract) signature result))
        tx)
    (rpc-error (e)
      (when (data e) (setf (decoded-data e) (decode-error (contract-data contract) (data e))))
      (error e))))
      
(defun decode-result (contract signature data)
  "Decode a function return value."
  (gethash "result" (jzon:parse (decode-parameters (arguments (gethash "outputs" (abi-entry contract signature))) data))))
  
(defun decode-error (contract data)
  "Decode a contract error."
  (let* ((hash (subseq data 2 10))
         (signature (or (hash->signature contract hash) "Error(string)"))
         (parameters (parameters contract signature))
         (data (subseq data 10)))
    (list signature (gethash "result" (jzon:parse (decode-parameters parameters data))))))
    
(defun hash->signature (contract hash)
  "Gets an entry's signature from a hash."
  (let ((hm (gethash "hashes" contract)))
    (loop for k being the hash-keys in hm using (hash-value v)
          do (when (string= v hash) (return-from hash->signature k))))
  ;;this is an error or an event - those aren't output in the abi's hashes section.
  (let ((signatures (signatures contract)))
    (find-if #'(lambda(x) (equal hash (subseq (ironclad:byte-array-to-hex-string (ironclad:digest-sequence :keccak/256 (ironclad:ascii-string-to-byte-array x))) 0 8))) signatures)))
```

The "client" `call` function first builds the `call-transaction` and associated data (most of the work is around encoding the parameters). `eth/call` is then called in order to get the result of that transaction. If the function call is read only, that is all we need. If not the client code needs to send the returned transaction to the chain for mining.

```
(defmethod send :before ((tx call-transaction))
  "Check that the contract call is not read-only."
  (when (read-only (contract-data (transaction-contract tx)) (slot-value tx 'signature))
    (error 'read-only :transaction tx)))
```

As you can see, we signal an error if the transaction is calling a `pure` or `view` function as there should be no need to mine that.

Finally, the last few bits we wanted to handle were logs.

```
(defmethod wait :after ((tx call-transaction))
  "Decode the logs from the transaction receipt."
  (let ((data (contract-data (transaction-contract tx)))
        (address (contract-address (transaction-contract tx))))
    (setf (slot-value tx 'logs) (decode-logs data address (coerce (gethash "logs" (transaction-receipt tx)) 'list)))))

(defun decode-logs (contract address logs)
  "Decode logs from a transaction receipt."
  (mapcar #'(lambda (x) (decode-log contract address x)) logs))

(defun decode-log (contract address log)
  "Decode a log entry. This handles indexed reference types too."
  (when (string= address (gethash "address" log))
    (let* ((topic (elt (gethash "topics" log) 0))
           (indexed (rest (coerce (gethash "topics" log) 'list)))
           (signature (hash->signature contract (subseq topic 2 10)))
           (inputs (coerce (gethash "inputs" (abi-entry contract signature)) 'list))
           (decoded-data (coerce (gethash "result" (jzon:parse (decode-parameters (format nil "(~{~a~^,~})" (map 'list #'(lambda (x) (gethash "type" x)) (remove-if-not #'(lambda (x) (not (gethash "indexed" x))) inputs))) (gethash "data" log)))) 'list)))
      (do ((inputs inputs (rest inputs))
           (current (car inputs) (cadr inputs))
           (indexed-index 0)
           (non-indexed-index 0)
           (result nil))
          ((null inputs) (cons signature (nreverse result)))
        (if (gethash "indexed" current)
            (progn
              (if (reference-p (gethash "type" current))
                  (push (nth indexed-index indexed) result) ; we cannot decode indexed reference types as they are hashes
                  (push (gethash "result" (jzon:parse (decode-parameters (gethash "type" current) (nth indexed-index indexed)))) result))
              (incf indexed-index))
            (progn
              (push (nth non-indexed-index decoded-data) result)
              (incf non-indexed-index)))))))
```

## Conclusion

This is the bulk of what we used for the now defunct Tinka platform prototype. It was a great learning experience to put this together, as I got to really drill down in the various specifications for transactions, encoding, etc.
