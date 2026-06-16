---
title: "Creating a NATS protocol parser in Haskell"
published: 2024-07-26
#author: "Kevin Hoffman"
description: "Creating a NATS protocol parser in Haskell"
tags: 
    - FP
    - Haskell
    - Parsers
---
_As an experiment (possibly in self-inflicted pain), I decided to write a NATS protocol parser in Haskell_.

It's hard to find excuses to build stuff in Haskell, so as I go through the process of trying to teach myself the basics, I've decided to build some things that I might actually use some day. In this case, it's the beginning of a NATS Haskell client. There are several existing clients out there, including two that are more than 8 years old. There is no "official" NATS Haskell client at the moment.

In typical functional programming style, I decided to work on the "pure" stuff first - defining the wire protocol for communicating with NATS. This involves deciding on my data types, and then writing a parser (input) and transformer (output) for that data.

According to the [NATS Client Protocol Spec](https://docs.nats.io/reference/reference-protocols/nats-protocol), we can define the known message types as shown in this snippet:

```haskell

data ProtocolMessage
  = InfoMessage
      { info :: Maybe ServerInfo,
        errorMessage :: Maybe String
      }
  | ConnectMessage
      { settings :: Maybe ConnectSettings,
        errorMessage :: Maybe String
      }
  | MsgMessage
      { subject :: String,
        reply :: Maybe String,
        sid :: String,
        len :: Integer,
        payload :: Maybe Text
      }
  | HeaderMsgMessage
      { subject :: String,
        reply :: Maybe String,
        sid :: String,
        headerBytes :: Integer,
        totalBytes :: Integer,
        messageHeaders :: [Header],
        payload :: Maybe Text
      }
  | PubMessage
      { subject :: String,
        reply :: Maybe String,
        len :: Integer,
        payload :: Maybe Text
      }
  | HeaderPubMessage
      { subject :: String,
        reply :: Maybe String,
        headerBytes :: Integer,
        totalBytes :: Integer,
        messageHeaders :: [Header],
        payload :: Maybe Text
      }
  | SubMessage
      { subject :: String,
        queueGroup :: Maybe String,
        sid :: String
      }
  | UnsubMessage
      { sid :: String,
        maxMessages :: Maybe Integer
      }
  | PingMessage
  | PongMessage
  | OkMessage
  | ErrMessage
      { reason :: String
      }
  deriving (Eq, Show)

data Header = Header {key :: String, value :: String} deriving (Eq, Show)
```
The `ServerInfo` and `ConnectSettings` types are just additional `data` that derive a JSON serializer and deserializer.

Being a filthy newb when it comes to Haskell, I followed some online recommendations that say for things like the NATS arbitrary payloads, one should use `Text`, and then if I need to convert into a `ByteString` later, it's just a matter of calling `encodeUtf8`.

An alternative design to this one would be to create a separate type for each of the messages rather than grouping them together in a _sum_ type like this. Personally, treating the protocol messages the way I would an `enum` in other languages just felt right.

## Parsing in Anger
Okay, so now that I've got my data types defined, it should be relatively easy to parse them, right?

_Narrator: It was not easy._

The first problem I had was the fact that there are _so many_ parsing libraries available in Haskell. Writing a parser (or "parser combinator" for the purists) seems to be one of the first things people do in this language. After many hours of research, I narrowed my choices down to:

* Use the [parsec](https://hackage.haskell.org/package/parsec) library
* Use the [megaparsec](https://hackage.haskell.org/package/megaparsec) library
* Roll my own (about 100 lines of code for the scaffolding)

I immediately ruled out the third option because it felt similar to the old addage _"never write your own crypto"_. There were as many websites and blog posts advocating for `parsec` as there were for `megaparsec`, and a neverending debate seemed constantly near boilover about which is best.

Ultimately I decided to use `megaparsec` because it's _mega_ (naturally). I felt like I made this decision by rolling a **D20** for initiative.

Parsers of this kind work by evaluating the rules you supply as the parser designer. This converts the input into something useful, and it then recursively calls all the other parsers until the input is exhausted, gradually building up more and more meaningful context. Each time a new parser in the "chain" is called, it's given both the output of the previous step and the remaining unparsed text.

When building parsers like this, I like to start at the smallest possible unit of input and then work my way out from there. This ends up being the ping and pong messages:

```haskell
pPingMessage :: Parser ProtocolMessage
pPingMessage = do
  _ <- string "PING" <* crlf
  return PingMessage

pPongMessage :: Parser ProtocolMessage
pPongMessage = do
  _ <- string "PONG" <* crlf
  return PongMessage
```
In the case of these parsers, they "eat" the string literals `PING` or `PONG` followed by a `crlf` (which is then discarded as per the `<*` operator).

The next up in terms of complexity might be the `UNSUB` message:

```haskell
pUnsubMessage :: Parser ProtocolMessage
pUnsubMessage = do
  _ <- string "UNSUB" <* hspace1
  sid <- many alphaNumChar
  maxMessages <- try (hspace1 *> fmap Just integer) <|> return Nothing
  _ <- crlf
  return UnsubMessage {..}
```

It took me _for ever_, plus annoying a bunch of folks online to figure out how to arrange the exact incantation to make this work. The first line should look familiar, as we "eat" the `UNSUB` string and then discard any whitespace following it. Next, we grab the subscription ID, which the spec defines as a _string_, so we use `many alphaNumChar` to capture that.

`maxMessages` is where I almost gave up. When parsers fail, they've already "eaten" their way down the failing path. So if we want to try a particular path that can fail, _or_ use that same data on an alternative, we need to use some of the monadic functions that do this.

```haskell
maxMessages <- try (hspace1 *> fmap Just integer) <|> return Nothing
```
Here we `try` to parse whitespace followed by something that can be parsed as an integer (note `integer` here is a _function_). If that fails, then we return `Nothing`, _and then backtrack to where the `try` block started_.

With all of that behind me and working, I moved on to messages that supported headers. In keeping with the inner-first building pattern, I defined how to parse a header:

```haskell
pHeaders :: Parser [Header]
pHeaders = do
  _ <- string "NATS/1.0" <* crlf
  many pHeader <* crlf

pHeader :: Parser Header
pHeader = do
  key <- validSubject <* char ':'
  value <- validSubject <* crlf
  return Header {..}
```
Per the spec, a list of headers is always followed by the `NATS/1.0` string literal and a cr/lf pair. Headers are relatively simple to parse because we can use the `:` as a separator token.

Now I can write a message parser for something that uses headers like `HMSG`:

```haskell
pHeaderMsgMessage :: Parser ProtocolMessage
pHeaderMsgMessage = do
  _ <- string "HMSG" <* hspace1
  subject <- validSubject <* hspace1 <?> "subject"
  sid <- many alphaNumChar <* hspace1 <?> "sid"
  (reply, headerBytes, totalBytes) <- try pHeaderMsgTailNoReply <|> pHeaderMsgTailReply
  messageHeaders <- pHeaders <?> "headers"
  payload <- optional $ do
    guard (totalBytes /= 0)
    takeP (Just "payload byte") (fromIntegral totalBytes) <* crlf
  return $ HeaderMsgMessage {..}
```
This one is pretty complicated. After eating `HMSG`, the subject, and the sid, we come to a branching point. Here we have to parse out a couple of bits of information like the header bytes, total bytes, and optional reply-to. Lastly, _if_ the total bytes number is greater than 0, we read exactly that number of bytes from input (followed by a discarded `crlf`).

Parsing can get _very_ frustrating with "false positives". The reply subject of a message can be _any_ string, which technically could be a string like "123". This means that "123" will satisfy both an integer parser and a string parser, and so my original naive implementations only worked for one branch, not both.

Instead of saying "if we have an integer, use it" (which can fail when someone supplies an integer-friendly reply-to subject!), we need to try and parse all the way to the end of the line. If we have a reply subject, header bytes, and total bytes, then that's one path. The other path is just header bytes and total bytes. We define these two backtrack-aware parsing paths as:

```haskell
pHeaderMsgTailNoReply :: Parser (Maybe string, Integer, Integer)
pHeaderMsgTailNoReply = do
  headerBytes <- integer <* hspace1 <?> "header bytes"
  totalBytes <- integer <* crlf <?> "total bytes"
  return (Nothing, headerBytes, totalBytes)

pHeaderMsgTailReply :: Parser (Maybe String, Integer, Integer)
pHeaderMsgTailReply = do
  reply <- fmap Just validSubject <* hspace1
  headerBytes <- integer <* hspace1 <?> "header bytes"
  totalBytes <- integer <* crlf <?> "total bytes"
  return (reply, headerBytes, totalBytes)
```
The `<?>` operator just lets me supply a label for that particular parse operation, which can help when parsing fails. When there's no reply subject, we return `(Nothing, headerBytes, totalBytes)`, and when there is one, we return `(reply, headerBytes, totalBytes)`.

Both of these branches parse the last 2 values on the line, but we have to set the backtrack position to before we read the subject. This parser could be _much_ simpler if the spec didn't allow reply subjects to be pure numbers, but alas this is where we are.

Next, we can do something a little bit different for the `CONNECT` and `INFO` messages, which both have literals immediately before a JSON payload.

```haskell
pConnectMessage :: Parser ProtocolMessage
pConnectMessage = do
  _ <- string "CONNECT" <* hspace1
  rest <- many printChar <* crlf
  case eitherDecode $ toByteString rest of
    Right a -> return ConnectMessage {settings = Just a, errorMessage = Nothing}
    Left e -> return ConnectMessage {settings = Nothing, errorMessage = Just e}
```
Here, `eitherDecode` is a function that we get from having derived a JSON decoder for the connection settings structure. If the decode fails, we return the `Left` side of either, and if it succeeds, we right the `Right` side. Technically the `Either` type lets you put whatever you want on whatever side, but the community convention seems to be that failures are `Left` while success is `Right`. This is obviously a conspiracy against left-handed people like me.

Now we're finally ready for the payoff. After having defined parsers for all of the individual message types, we can create a root parser that uses alternate (`<|>`) syntax to make it obvious how message parsing works:

```haskell
parseMessage :: Parser ProtocolMessage
parseMessage =
  pMsgMessage
    <|> pInfoMessage
    <|> pConnectMessage
    <|> pPubMessage
    <|> pSubMessage
    <|> pUnsubMessage
    <|> pHeaderPubMessage
    <|> pHeaderMsgMessage
    <|> pOkMessage
    <|> pErrMessage
    <|> pPingMessage
    <|> pPongMessage
```
The order matters here. We need to be very explicit about how we define the individual parsers so that we don't accidentally try and parse one input as a different type. Fortunately, we don't need to do any backtracking here because each NATS message has a unique prefix. This means our parsers will all fail when reading an incorrect first token (e.g. `HMSG`, `MSG`, `INFO`, etc). The monadic portion of this operator enables us to chain all these together, knowing that if one fails, we will attempt the next one in line. If all of them fail, then `parseMessage` will return a `Left a` where `a` is the parser error type.

I really disliked the process of learning how to build this parser. I personally am not a big fan of DSLs, and so the DSL-like nature of how all these parser combinators, er, _combine_, rubbed me the wrong way. 

It feels like there's "mainstream" Haskell, and then there's _parsing_, which is a completely different animal. I will admit, however, that after all the suffering and smashing my head against the desk, I came to like how testable and verifiable these parsers are.

Here's an example of test scaffolding for the `PING` message:

```haskell

spec :: Spec
spec = do
  doParserCases
  doTransformerCases  

parserCases :: [(Text, ProtocolMessage)]
parserCases = [("PING\r\n", PingMessage)]

transformerCases :: [(ProtocolMessage, Text)]
transformerCases = map (\(a,b) -> (b,a)) parserCases

doParserCases :: SpecWith ()
doParserCases = parallel $ do
  describe "PING parser" $ do
    forM_ parserCases $ \(input, want) -> do
      it (printf "correctly parses explicit case %s" (show input)) $ do
        let output = parse parseMessage "" input
        output `shouldBe` Right want

doTransformerCases :: SpecWith ()
doTransformerCases = parallel $ do
  describe "PING transformer" $ do
    forM_ transformerCases $ \(input, want) -> do
      it (printf "correctly transforms %s" (show input)) $ do
        transform input `shouldBe` encodeUtf8 want
```
You can see how easy it is to reuse this format to define the expected inputs, what they should parse as, and to ensure that they are output (transformed) exactly as they were input.

In other words, I don't think learning how to build parsers teaches you the fundamentals of Haskell. However, having a very good working knowledge of type classes, monads, and applicatives can help make sense of all this stuff that seemed like gibberish when I first started.

I am not claiming to have figured this all out on my own. As usual, learning is a community effort and I got a lot of help from Reddit, Discord, and got a lot of project layout inspiration from [natskell](https://github.com/samisagit/natskell/tree/main), one of the other Haskell NATS clients.

Next up, I need to add some basic network connectivity to talk to the NATS server. I'm sure that'll be easy, right?
