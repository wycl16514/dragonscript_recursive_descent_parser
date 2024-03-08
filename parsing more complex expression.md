we can enhance our parser to handle more complex expression. This time we add support for key words like true, false and nil, first we create test case for it:
```js
it("should parse expression with +, -, !, operator and key words", () => {
        let parser = new RecursiveDescentParser("!true;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("!false;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("false + !true;")
        expect(parser.parse).not.toThrow()
        /*
        what this could be? !false evalue to true, then -true evalue to -1,
        what dose it mean adding string with -1? in js it will turn -1 into
        string "-1" and connocate the two string together, then
        "hello" + -!false -> "hello-1", in other language it might be error
        */
        parser = new RecursiveDescentParser('"hello" + -!false;')
        expect(parser.parse).not.toThrow()

        parser = new RecursiveDescentParser('"hello" + -!nil;')
        expect(parser.parse).not.toThrow()
    })

```
the last case is interesting, as metioned in the comment, it might be an error for many languages that's why we only construct ast in parser and split out the
evaluation of the ast from the parser.Let's add code to make the test case passed, it turns out quit simple to satisfy the case, we only need to add keyword 
support to primary function:
```js
 primary = (parentNode) => {
        //primary -> NUMBER | STRING | "true" | "false" | "nil" | "(" expression ")" | epsilon
        const token = this.matchTokens([Scanner.NUMBER, Scanner.STRING,
        Scanner.TRUE, Scanner.FALSE, Scanner.NIL])
        if (token === null) {
            //primary -> epsilon
            return false
        }
    ...
}
```
After adding TRUE, FALSE, NIL to the matchTokens in primary, the above test case can be passed. Now let's add support for operator * and /, here is the test 
case:
```js
 it("should support operator * and / in expression", () => {
        let parser = new RecursiveDescentParser("1*2+3;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("4/2 - 5;")
        expect(parser.parse).not.toThrow()
    })
```
We need to change the code in factorRecursive because this function handle the supporting of the two operators:
```js
 factorRecursive = (parentNode) => {
        const opToken = this.matchTokens([Scanner.START, Scanner.SLASH])
        if (opToken === null) {
            //factor_recursive -> epsilon
            return
        }
        //factor_recursive ->  ("*|"/") factor
        const factorRecursiveNode = this.createParseTreeNode("factor_recursive")
        factorRecursiveNode.attributes = {
            value: opToken.lexeme,
            token: opToken,
        }
        parentNode.children.push(factorRecursiveNode)
        this.advance()
        this.factor(factorRecursiveNode)
    }
```
After adding the code above we can make the newly added test case to pass, let's add support for brackets, here is the test case:
```js
it("should support brackets in expression", () => {
        let parser = new RecursiveDescentParser("1*2+(4-3);")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("!false + (8-2)/3;")
        expect(parser.parse).not.toThrow()
    })
```
we need to change many places to make the test case pass:
```js
equalityRecursive = (parentNode) => {
        const opToken = this.matchTokens([Scanner.BANG_EQUAL, Scanner.EQUAL_EQUAL])
        if (!opToken) {
            return
        }
        //equality_recursive -> ("!="|"==") equality
        throw new Error("equalityRecursive TODO")
    }

comparisonRecursive = (parentNode) => {
        const opToken = this.matchTokens([Scanner.LESS_EQUAL, Scanner.LESS,
        Scanner.GREATER, Scanner.GREATER_EQUAL])
        if (!opToken) {
            //comprison -> epsilon
            return
        }
        //comparison -> epsilon | (">" | ">=" | "<" | "<=") comparison
        throw new Error("comparisonRecursive TODO")
    }

 primary = (parentNode) => {
        //primary -> NUMBER | STRING | "true" | "false" | "nil" | "(" expression ")" | epsilon
        const token = this.matchTokens([Scanner.NUMBER, Scanner.STRING,
        Scanner.TRUE, Scanner.FALSE, Scanner.NIL, Scanner.LEFT_PAREN])
      ...
       //primary -> "(" expression ")"
        if (token.token === Scanner.LEFT_PAREN) {
            this.expression(primary)
            if (!this.matchTokens([Scanner.RIGHT_PAREN])) {
                throw new Error("Miss matching ) in expression")
            }
            //scann over )
            this.advance()
        }
        return true
```
After adding the above code, the newly added test case can be passed
