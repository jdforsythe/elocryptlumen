# Eloquent Encryption/Decryption for Lumen 5

Automatically encrypt and decrypt Lumen 5 Eloquent values.

## READ THIS FIRST

Encrypted values are usually longer than plain text values.  Sometimes much longer.  You
may find that the column widths in your database tables need to be extended to store the
encrypted values.

If you are encrypting long strings such as JSON blobs then the encrypted values may be
longer than a VARCHAR field can support, and you may need to extend your column types to
TEXT or LONGTEXT.

## What Does This Do?

This encrypts and decrypts columns stored in database tables in Lumen applications
transparently, by encrypting data as it is stored in the model attributes and decrypting
data as it is recalled from the model attributes.

All data that is encrypted is prefixed with a tag (currently `__ELOCRYPT__;`) so that
encrypted data can be easily identified.

This supports columns that store either encrypted or non-encrypted data to make migration
easier.  Data can be read from columns correctly regardless of whether it is encrypted or
not but will be automatically encrypted when it is saved back into those columns.

## Contributors

This is Del's Laravel 5 Elocrypt package ported so it can be used in Lumen without enabling
facades. I have made the following changes:

* Load the `'encrypter'` directly instead of through the `Crypt` facade, enabling use in Lumen
  without turning on Facades

* Enable setting the elocrypt prefix in the `.env` file (`ELOCRYPT_PREFIX=`)

The original Laravel 5 package is here: https://github.com/delatbabel/elocrypt

The original Laravel 4 package is here: https://github.com/dtisgodsson/elocrypt

## Installation

This package can be installed via Composer by adding the following to your `composer.json` file:

```
    "require": {
        "jdforsythe/elocryptlumen": "~1.0"
    }
```

You must then run the following command:

```
    composer update
```

## Usage

Simply reference the Elocrypt trait in any Eloquent Model you wish to apply encryption to and define
an `$encrypts` array containing a list of the attributes to encrypt.

For example:

```php
    use Jdforsythe\ElocryptLumen\Elocrypt;

    class User extends Eloquent {

        use Elocrypt;
       
        /**
         * The attributes that should be encrypted on save.
         *
         * @var array
         */
        protected $encrypts = [
            'address_line_1', 'first_name', 'last_name', 'postcode'
        ];
    }
```

You can combine `$casts` and `$encrypts` to store encrypted arrays.  An array will first be converted to JSON
and then encrypted.

For example:

```php
    use Jdforsythe\ElocryptLumen\Elocrypt;

    class User extends Eloquent {

        use Elocrypt;

        protected $casts = ['extended_data' => 'array'];

        protected $encrypts = ['extended_data'];
    }
```

# How it Works?

By including the Elocrypt trait, the setAttribute() and getAttributeFromArray() methods provided
by Eloquent are overridden to include an additional step. This additional step simply checks
whether the attribute being set or get is included in the `$encrypts` array on the model,
and either encrypts/decrypts it accordingly.

## Summary of Methods in Illuminate\Database\Eloquent\Model

This surveys the major methods in the Eloquent Model class as of
Laravel v 5.1.12 and checks to see how those models set attributes
and hence how they are affected by this trait.

* constructor -- calls fill()
* fill() -- calls setAttribute() which has been extended to encrypt the data.
* hydrate() -- TBD
* create() -- calls constructor and hence fill()
* firstOrCreate -- calls constructor
* firstOrNew -- calls constructor
* updateOrCreate -- calls fill()
* update() -- calls fill()
* toArray() -- calls attributesToArray()
* jsonSerialize() -- calls toArray()
* toJson() -- calls toArray()
* attributesToArray() -- calls getArrayableAttributes().
* getAttribute -- calls getAttributeValue()
* getAttributeValue -- calls getAttributeFromArray()
* getAttributeFromArray -- calls getArrayableAttributes()
* getArrayableAttributes -- has been extended here to decrypt the data.
* setAttribute -- has been extended here to encrypt the data.
* getAttributes -- has been extended here to decrypt the data.

## Keys and IVs

The key for the Lumen Encrypter service is set in the `.env` file as `APP_KEY`. Lumen defaults to AES-256-CBC
cipher.

The IV for encryption is randomly generated.

# FAQ

## Manually Encrypting Data

You can manually encrypt or decrypt data using the `encryptedAttribute()` and `decryptedAttribute()` functions.
An example is as follows:

```php
    $user = new User();
    $encryptedEmail = $user->encryptedAttribute(Input::get("email"));
```

## Encryption and Searching

You will not be able to search on encrypted data, because it is encrypted.  Comparing encrypted values
would require a fixed IV which introduces security issues.

If you need to search on data then either:

* Leave it unencrypted, or
* Hash the data and search on the hash instead of the encrypted value.  Use a well known hash algorithm
  such as SHA256.

You could store both a hashed and an encrypted value, use the hashed value for searching and retrieve
the encrypted value for other uses.

## Encryption and Authentication

The same problem with searching applies for authentication because authentication requires a user search.

If you have an authentication table where you encrypt the user data including the login data (for example the email),
this will prevent Auth::attempt from working.  For example this code will not work:

```php
    $auth = Auth::attempt(array(
                "email"     =>  Input::get("email"),
                "password"  =>  Input::get("password"),
    ), $remember);
```

As for searching, comparing the encrypted email will not work, because it would require a fixed IV
which introduces security issues.

What you will need to do instead is to hash the email address using a well known hash function (e.g.
SHA256 or RIPE-MD160) rather than encrypt it, and then in the Auth::attempt function you can compare
the hashes.

If you need access to the email address then you could store both a hashed and an encrypted email
address, use the hashed value for authentication and retrieve the encrypted value for other uses
(e.g. sending emails).
