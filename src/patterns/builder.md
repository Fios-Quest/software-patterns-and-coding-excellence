Builder
=======

The builder pattern allows us to construct complex types in steps.

For example, imagine we have a 'User' struct that holds a lot of information on a user. Some of that information is
required and some of it is not.

```rust
use std::collections::HashSet;

# // shush shush shush this code is supposed to look good, not be good
# struct Username(String);
# impl From<String> for Username {
#     fn from(s: String) -> Self  {
#         Self(s)
#     }
# }
# #[derive(Eq, PartialEq, Hash)]
# struct EmailAddress(String);
# impl From<String> for EmailAddress {
#     fn from(s: String) -> Self  {
#         Self(s)
#     }
# }
# struct PhoneNumber(String);
# impl From<String> for PhoneNumber {
#     fn from(s: String) -> Self  {
#         Self(s)
#     }
# }
# 
struct User {
    // Required valus
    username: Username,
    primary_email: EmailAddress,
    // Optional values
    secondary_emails: HashSet<EmailAddress>,
    phone_number: Option<PhoneNumber>,
}
```

There are arguably two versions of this pattern, a simpler version that will work in just about any language that I'm
going to refer to as "Builder Lite", and a more complex version that only works in languages with strict generic typing,
the "[Typestate](./typestate.md) Builder". Each has their own pros and cons, and we'll go over those too.

Builder Lite
------------

The traditional builder pattern uses a type that mirrors the type you want to build but everything is optional. We
use methods to update each field, and then have a finalizer that takes the data we've stored in the builder and attempts
to convert it into the target type.

For our `User` example that might look like this.


```rust
use std::error::Error;
use std::collections::HashSet;

# // shush shush shush this code is supposed to look good, not be good
# struct Username(String);
# impl<S: ToString> From<S> for Username {
#     fn from(s: S) -> Self  {
#         Self(s.to_string())
#     }
# }
# #[derive(Eq, PartialEq, Hash)]
# struct EmailAddress(String);
# impl<S: ToString> From<S> for EmailAddress {
#     fn from(s: S) -> Self  {
#         Self(s.to_string())
#     }
# }
# struct PhoneNumber(String);
# impl From<String> for PhoneNumber {
#     fn from(s: String) -> Self  {
#         Self(s)
#     }
# }
# 
# struct User {
#     // Required valus
#     username: Username,
#     primary_email: EmailAddress,
#     // Optional values
#     secondary_emails: HashSet<EmailAddress>,
#     phone_number: Option<PhoneNumber>,
# }
#
enum UserBuilderError {
    NoUsername,
    NoPrimaryEmail,
}


#[derive(Default)]
struct UserBuilder {
    username: Option<Username>,
    primary_email: Option<EmailAddress>,
    secondary_emails: HashSet<EmailAddress>,
    phone_number: Option<PhoneNumber>,
}

impl UserBuilder {
    fn new() -> Self {
        Self::default()
    }

    fn with_username(mut self, username: Username) -> Self {
        self.username = Some(username);
        self
    }

    fn with_primary_email(mut self, email: EmailAddress) -> Self {
        self.primary_email = Some(email);
        self
    }

    fn add_secondary_email(mut self, email: EmailAddress) -> Self {
        self.secondary_emails.insert(email);
        self
    }

    fn with_phone_number(mut self, phone_number: PhoneNumber) -> Self {
        self.phone_number = Some(phone_number);
        self
    }

    fn build(self) -> Result<User, UserBuilderError> {
        let username = self.username.ok_or(UserBuilderError::NoUsername)?;
        let primary_email = self.primary_email.ok_or(UserBuilderError::NoPrimaryEmail)?;
        let secondary_emails = self.secondary_emails;
        let phone_number = self.phone_number;

        Ok (User {
            username,
            primary_email,
            secondary_emails,
            phone_number,            
        })
    }
}

fn main () {
    // We can successfully build a User if we have all the required information
    let user_result = UserBuilder::new()
        .with_username(Username::from("Fio"))
        .with_primary_email(EmailAddress::from("fio@example.com"))
        .build();
    assert!(user_result.is_ok());

    // But if we don't give all the required information we get an error
    let user_result = UserBuilder::new()
        .with_username(Username::from("Fio"))
        .build();
    assert!(user_result.is_err());
}
```

Typestate Builder
-----------------

In the previous example we need to deal with calling `.build()` on a builder that may not have all the required
information. To manage this potential problem we return a result. What if I told you, we can write this code in such
a way as to be sure that the `.build()` method can only be used once we can guarantee we have everything we need, thus
negating the Result requirement.

This is an advanced application of the [Typestate](./typestate.md) pattern. Instead of migrating between concrete
types representing individual states, we can use generics as markers on top of which we can implement different methods

The only slight trick is that generic types must be "used" _in_ our type. For example, the following won't compile 
because we didn't use "T" in the struct itself, even though our instantiation uses a Unit Struct:

```rust,compile_fail
struct BadExample<T> {
    data: String,
}

struct Marker;

# fn main() {
let example = BadExample::<Marker> { data: "This won't work".to_string() };
# }
```

This is where `PhantomData` comes in. It's a zero-sized marker that "uses" the types in generics, allowing you to use
generics as nothing more than a compile time marker.

```rust
use std::marker::PhantomData;

struct GoodExample<T> {
    data: String,
    marker_for_t: PhantomData<T>,
}

struct Marker;

# fn main() {
let example = GoodExample::<Marker> {
    data: "This will work!".to_string(),
    marker_for_t: PhantomData,
};
# }
```

Let's build our builder again using this method.

```rust
use std::error::Error;
use std::collections::HashSet;
use std::marker::PhantomData;

# // shush shush shush this code is supposed to look good, not be good
# struct Username(String);
# impl<S: ToString> From<S> for Username {
#     fn from(s: S) -> Self  {
#         Self(s.to_string())
#     }
# }
# #[derive(Eq, PartialEq, Hash)]
# struct EmailAddress(String);
# impl<S: ToString> From<S> for EmailAddress {
#     fn from(s: S) -> Self  {
#         Self(s.to_string())
#     }
# }
# struct PhoneNumber(String);
# impl From<String> for PhoneNumber {
#     fn from(s: String) -> Self  {
#         Self(s)
#     }
# }
# 
# struct User {
#     // Required valus
#     username: Username,
#     primary_email: EmailAddress,
#     // Optional values
#     secondary_emails: HashSet<EmailAddress>,
#     phone_number: Option<PhoneNumber>,
# }
#
# enum UserBuilderError {
#     NoUsername,
#     NoPrimaryEmail,
# }
# 

// We'll use these unit structs as markers for when values are set
// Deriving Default means we can simplify our constructor
#[derive(Default)]
struct Set;
#[derive(Default)]
struct Unset;

#[derive(Default)]
struct UserBuilder<U, PE> {
    username: Option<Username>,
    primary_email: Option<EmailAddress>,
    secondary_emails: HashSet<EmailAddress>,
    phone_number: Option<PhoneNumber>,
    // We need to record the markers somewhere but this won't impact runtime
    username_set: PhantomData<U>,
    primary_email_set: PhantomData<PE>,
}

// The constructor only exists for UserBuilder<Unset, Unset> so that's how our
// builder starts
impl UserBuilder<Unset, Unset> {
    fn new() -> UserBuilder<Unset, Unset> {
        Self::default()
    }
}

impl<U, PE> UserBuilder<U, PE> {
    // When we set the username we have to completely migrate the type and
    // create a new struct for the generics to work.
    fn with_username(mut self, username: Username) -> UserBuilder<Set, PE> {
        UserBuilder {
            username: Some(username),
            primary_email: self.primary_email,
            secondary_emails: self.secondary_emails,
            phone_number: self.phone_number,
            username_set: PhantomData,
            primary_email_set: PhantomData,
        }
    }

    // Same goes for primary email
    fn with_primary_email(mut self, email: EmailAddress) -> UserBuilder<U, Set> {
        UserBuilder {
            username: self.username,
            primary_email: Some(email),
            secondary_emails: self.secondary_emails,
            phone_number: self.phone_number,
            username_set: PhantomData,
            primary_email_set: PhantomData,
        }
    }

    // The other setters return the same type including generics, whatever they were before
    fn add_secondary_emaail(mut self, email: EmailAddress) -> UserBuilder<U, PE> {
        self.secondary_emails.insert(email);
        self
    }

    fn with_phone_number(mut self, phone_number: PhoneNumber) -> UserBuilder<U, PE> {
        self.phone_number = Some(phone_number);
        self
    }
}

// By only implementing `.build()` on UserBuilder<Set, Set>, the method will only
// be available once both `.set_username()` and `.set_primary_email()` are called.
impl UserBuilder<Set, Set> {
    fn build(self) -> User {
        let username = self.username
            .expect("It should not be possible to call build with username unset");
        let primary_email = self.primary_email
            .expect("It should not be possible to call build with primary email unset");
        let secondary_emails = self.secondary_emails;
        let phone_number = self.phone_number;

        User {
            username,
            primary_email,
            secondary_emails,
            phone_number,
        }
    }
}

fn main () {
    // We can only build a User if we have all the required information
    let user = UserBuilder::new()
        .with_username(Username::from("Fio"))
        .with_primary_email(EmailAddress::from("fio@example.com"))
        .build();

    // This won't compile because .build() only exists on UserBuilder<Set, Set>
    // let user_result = UserBuilder::new()
    //     .set_username(Username::from("Fio"))
    //     .build();
}
```

Using the typestate builder we prevent `.build()` being called unless all required data has been set. Rather than
runtime validation, we get compile time validation!

Pro's and Con's
---------------

I don't think there's one "right" builder to use.

Having compile time validation that you've used the builder correctly is nice, but it adds a lot of complexity through
the type system, and you can arguably be sure you've used the builder correctly through tests. But using the lite
builder adds more complexity to your tests and error handling code.

Its also worth noting that the typestate builder won't work if you can't guarantee each method is called at runtime. 
Methods that set a required parameter change the type of the builder, meaning you can't put a call in a branch (such as
an if) because you can't reconcile the types later.
