we will develop a recursive descent parser with the following grammar:

statement -> expression SEMI

expression -> equality

equality -> comparison equality_recursive

equality_recursive-> ℇ | (“！=”| “==” ) equality

comparison → term  comparison_recursive

comparison_recursive-> ℇ | ( ">" | ">=" | "<" | "<=" ) comparison

term → factor term_recursive

term_recursive -> ℇ | (“-”|”+”) term

factor -> unary factor_recursive

factor_recursive-> ℇ | (“/”|”*”) factory

unary →  primary unary_recursive

unary_recursive -> ℇ | (“!”|”-”) unary

primary → NUMBER | STRING | "true" | "false" | "nil"  |  "(" expression  ")" | ℇ 


let's let a new file named recursive_descent_parser.js in the folder of parser, first we construct the framework for our parser:
```js
import Scanner from '../../scanner/token'
export default class RecursiveDescentParser {
    constructor(expression) {
        this.expression = expression
        this.scanner = new Scanner(expression);
        /*
        save all scanned tokens in the array, we can get previous scanned 
        token by using index
        */
        this.tokens = []
        this.current = -1
        //contains root for each arithmetic express
        this.parseTrees = []
    }
}
```
then we create a parser.test.js file and write down the test cases for our parser beforehand:
```js
import RecursiveDescentParser from './recursive_descent_parser'
import Scanner from '../../scanner/token'
describe("Testing token scanning from parser", () => {
    it("should get correct tokens by advance", () => {
        const parser = new RecursiveDescentParser("1;")
        const numToken = parser.getToken()
        expect(numToken).toMatchObject({
            lexeme: "1",
            token: Scanner.NUMBER,
            line: 0,
        })
        parser.advance()
        const semiToken = parser.getToken()
        expect(semiToken).toMatchObject({
            lexeme: ";",
            token: Scanner.SEMICOLON,
            line: 0,
        })
    });

})
```
because when running command "npm test", react js will execute all test cases in all *.test.js files, therefore I make a little trick, change scanner.test.js to scanner.test1.js, then
we can avoid wasting time on executing test cases in this file, and then run the test command to execute the test case above and make sure it fail, then we add the following code to 
satisfy the test case:
```js  
    getToken = () => {
        return this.tokens[this.current]
    }

    advance = () => {
        if (this.current + 1 >= this.tokens.length) {
            /*
            if current at the end of array, we push a new token
            otherwise there has token ahead of current ,then we just 
            return the next already existing token
            */
            const token = this.scanner.scan()
            if (token.token !== Scanner.EOF) {
                this.tokens.push(token)
                this.current += 1
            }
        }
    }
```
adding the above code can let the previous failed test case to pass. Now we add second test case for testing whether the parser can shift to previous token:
```js
 it("should get correct token by previous", () => {
        const parser = new RecursiveDescentParser("1;")
        parser.advance()
        parser.advance()
        parser.previous()
        const numToken = parser.getToken()
        expect(numToken).toMatchObject({
            lexeme: "1",
            token: Scanner.NUMBER,
            line: 0,
        })
    })
```
because we havn't add the previous function to parser, the test case would fail, then we can add code to pass it:
```js
    previous = () => {
        if (this.current > 0) {
            this.current -= 1
        }
    }
```
adding the above code can make sure the second case can be passed. We add the following three test cases and make sure they can be passed normally:
```js
 it("should get the last token when calling advance with times more than existing tokens", () => {
        const parser = new RecursiveDescentParser("1;")
        parser.advance()
        parser.advance()
        parser.advance()
        parser.advance()
        const semiToken = parser.getToken()
        expect(semiToken).toMatchObject({
            lexeme: ";",
            token: Scanner.SEMICOLON,
            line: 0,
        })
    })

    it("should get the beginning token when calling previous with times more than existing tokens", () => {
        const parser = new RecursiveDescentParser("1;")
        parser.advance()
        parser.previous()
        parser.previous()
        parser.previous()
        const numToken = parser.getToken()
        expect(numToken).toMatchObject({
            lexeme: "1",
            token: Scanner.NUMBER,
            line: 0,
        })
    })

    it("should get the first token by calling previous withou calling advance at before", () => {
        const parser = new RecursiveDescentParser("1;")
        parser.previous()
        const numToken = parser.getToken()
        expect(numToken).toMatchObject({
            lexeme: "1",
            token: Scanner.NUMBER,
            line: 0,
        })
    })
```
We can begin our parsing process now, first we add a test case for parsing make sure it fail then we can add code to satisfy it:
```js
describe("Testing creation of abstract syntax tree ", () => {
    it("should parse expression with only number or string", () => {
        let parser = new RecursiveDescentParser("1;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser('"hello world";')
        expect(parser.parse).not.toThrow()
    })

})
```
we havn't add the parse function to our parser, the test case is sure to fail, first let's add some helper function first:
```js
createParseTreeNode = (name) => {
        return {
            name: name,
            children: [],
            attributes: "",
        }
    }

    matchTokens = (tokens) => {
        //check the given token can match the given tokens or not
        const curToken = this.getToken()
        for (let i = 0; i < tokens.length; i++) {
            if (curToken.token == tokens[i]) {
                return curToken
            }
        }

        return null
    }
```
createParseTreeNode is still the same, but matchTokens will match an array of tokens, it will get the current token and loop over the passed token array, if one of them can match the current token, it 
will return the matched token, otherwise it will return null, now let's add the grammar rule implementation:
```js
parse = () => {
        //clear the parsing tree
        const treeRoot = this.createParseTreeNode("root")
        //clear all
        this.parseTrees = []
        //execute the first rule
        this.statement()
        treeRoot.children = this.parseTrees
        return treeRoot
    }

    statement = () => {
        const stmtNode = this.createParseTreeNode("stmt")
        //stmt -> expression SEMI
        this.expression(stmtNode)
        const token = this.matchTokens([Scanner.SEMICOLON])
        if (token === null) {
            throw new Error("statement miss matching SEMICOLON")
        }
        this.parseTrees.push(stmtNode)
    }

    expression = (parentNode) => {
        //expression -> equality
        const exprNode = this.createParseTreeNode("expression")
        this.equality(exprNode)
        parentNode.children.push(exprNode)
    }

    equality = (parentNode) => {
        //equality -> comparison equality_recursive
        const equNode = this.createParseTreeNode("equality")
        this.comparison(equNode)
        this.equalityRecursive(equNode)
        parentNode.children.push(parentNode)
    }

    comparison = (parentNode) => {
        //comparison -> term comparison_recursive
        const compaNode = this.createParseTreeNode("comparison")
        this.term(compaNode)
        this.comparisonRecursive(compaNode)
        parentNode.children.push(compaNode)
    }

    equalityRecursive = (parentNode) => {
        if (this.matchTokens([Scanner.SEMICOLON])) {
            return
        }
        //equality_recursive -> ("!="|"==") equality
        throw new Error("TODO")
    }

    comparisonRecursive = (parentNode) => {
        if (this.matchTokens([Scanner.SEMICOLON])) {
            return
        }
        //comparison -> epsilon | (">" | ">=" | "<" | "<=") comparison
        throw new Error("TODO")
    }

    term = (parentNode) => {
        //term -> factor term_recursive
        const term = this.createParseTreeNode("term")
        this.factor(term)
        this.termRecursive(term)
        parentNode.children.push(term)
    }

    termRecursive = (parentNode) => {
        if (this.matchTokens([Scanner.SEMICOLON])) {
            return
        }
        //term_recursive -> epsilon | ("-" | "+")
        throw new Error("TODO")
    }

    factor = (parentNode) => {
        //factor -> unary factor_recursive
        const factor = this.createParseTreeNode("factor")
        this.unary(factor)
        this.factorRecursive(factor)
        parentNode.children.push(factor)
    }

    factorRecursive = (parentNode) => {
        if (this.matchTokens([Scanner.SEMICOLON])) {
            return
        }
        //factor_recursive -> epsilon | ("*" "/") factor
        throw new Error("TODO")
    }

    unary = (parentNode) => {
        //unary -> primary | unary_recursive 
        const unary = this.createParseTreeNode("unary")
        this.primary(unary)
        this.unaryRecursive(unary)
        parentNode.children.push(unary)
    }

    unaryRecursive = (parentNode) => {
        //unary_recursive -> epsilon | unary
        if (this.matchTokens([Scanner.SEMICOLON])) {
            return
        }

        throw new Error("TODO")
    }

    primary = (parentNode) => {
        //primary -> NUMBER | STRING | "true" | "false" | "nil" | "(" expression ")" | epsilon
        const token = this.matchTokens([Scanner.NUMBER, Scanner.STRING])
        if (token !== null) {
            this.advance()
        } else {
            //primary -> epsilon
            return
        }

        const primary = this.createParseTreeNode("primary")
        primary.attributes = {
            value: token.lexeme,
            token: token,
        }
        parentNode.children.push(primary)
    }

```
the parsing process is just turn those rules into function calls, notice that in primary, if we havn't match given tokens, it will return directly, this is equivalence to the rule primary->epsilon, any 
rule that matching epsilon will do the same, and every function with suffix "Recursive" will throw an exception if the current token can't match semicolon, because our test case don't need to call into 
them, the error throwing will help us to make newly add test cases fail.

After adding the above code we can make sure our test case can pass.Now let's add more test cases and enhance our parser:
```js
it("should parse expression with + and - operatior", () => {
        let parser = new RecursiveDescentParser("1+2;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser('"hello" + "world";')
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser('4 - 3;')
        expect(parser.parse).not.toThrow()
    })
```
The case will fail because some parsing functions related to parsing process throw exception, now we can change those functions and make the test case passed:
```js
    termRecursive = (parentNode) => {
        const opToken = this.matchTokens([Scanner.MINUS, Scanner.PLUS])
        if (opToken === null) {
            //term_recursive -> epsilon
            console.log("term recursive epsilon")
            return
        }
        //term_recursive ->  ("-" | "+") term
        let nodeName = ""
        if (opToken.token === Scanner.MINUS) {
            nodeName = "MINUS"
        } else {
            nodeName = "ADD"
        }
        const opNode = this.createParseTreeNode(nodeName)
        opNode.attributes = {
            value: opToken.lexeme,
            token: opToken,
        }
        parentNode.children.push(opNode)
        this.advance()
        this.term(opNode)
    }

factorRecursive = (parentNode) => {
        const opToken = this.matchTokens([Scanner.START, Scanner.SLASH])
        if (opToken === null) {
            //factor_recursive -> epsilon
            return
        }
        //factor_recursive ->  ("*" "/") factor
        throw new Error("factorRecursive TODO")
    }

 unary = (parentNode) => {
        //unary -> primary | unary_recursive 
        /*
        we need to choose one of the two rules, which one should we choose?
        if current token can be matched in primary then choose primary,
        otherwise if the current token is ! or -, then choose unary_recursive
        */
        const unaryNode = this.createParseTreeNode("unary")
        if (this.primary(unaryNode) === false) {
            this.unaryRecursive(unaryNode)
        }
        parentNode.children.push(unaryNode)
    }

    unaryRecursive = (parentNode) => {
        //unary_recursive -> epsilon |("!"|"-") unary
        const opToken = this.matchTokens([Scanner.BANG, Scanner.MINUS])
        if (opToken === null) {
            //unary_recursive -> epsilon 
            return
        }

        throw new Error("unaryRecursive TODO")

    }

    primary = (parentNode) => {
        //primary -> NUMBER | STRING | "true" | "false" | "nil" | "(" expression ")" | epsilon
        const token = this.matchTokens([Scanner.NUMBER, Scanner.STRING])
        if (token === null) {
            //primary -> epsilon
            return false
        }
        const primary = this.createParseTreeNode("primary")
        primary.attributes = {
            value: token.lexeme,
            token: token,
        }
        parentNode.children.push(primary)
        this.advance()
        return true
    }
```
we need to pay attention to several points, first the parsing of 1+2; "hello"+"world"; and 4-3 involes rules that are term_recursive, factor_recursive, 
unary, unary_recursive, primary and we need to modify their conrresponding functions. Second notice code in unary, because the rule:
unary->primary | unary_recursive
this rule need to choose one from two rules(primary and unary_recursive), but how can we decide which one to choose? 

we check which rule can consume the currenttoken, if the current token can consumed by primary, that is the current token is NUMBER or STRING, then we choose rule primary, otherwise we choose 
unary_recursive The third point is that we change the content of the error being threw, we add the function name in the error content, which can help us know which function cause the test case to fail. 

After adding the above code we can make sure the newly added test case can be passed.Let's add one more test case:
```js
 it("should parse expression with + or - and negative number", () => {
        let parser = new RecursiveDescentParser("-1+2;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("1--2;")
        expect(parser.parse).not.toThrow()
    })
```
we need to enhance the parser enable it to handle negative number in expression, let's do the code as following:
```js
it("should parse expression with + or - and number with unary operator prefix", () => {
        let parser = new RecursiveDescentParser("-1+2;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("1--2;")
        expect(parser.parse).not.toThrow()
        parser = new RecursiveDescentParser("!1-!-2;")
        expect(parser.parse).not.toThrow()
    })
```
you can see in the test case, we have quit complex expression that is !1-!-2, the !1 is negate of 1 which results to false, and !-2 is negate of -2 which results to true, if we can make our parser passes
this case then the parser is quit powerful, let's have the following code to make it passed:
```js
unaryRecursive = (parentNode) => {
        //unary_recursive -> epsilon |("!"|"-") unary
        const opToken = this.matchTokens([Scanner.BANG, Scanner.MINUS])
        if (opToken === null) {
            //unary_recursive -> epsilon 
            return
        }

        //unary_recursive -> ("!"|"-") unary
        let opName = ""
        if (opToken.token === Scanner.BANG) {
            opName = "BANG"
        } else {
            opName = "MINUS"
        }

        const unaryRecursiveNode = this.createParseTreeNode("unary_recursive")
        unaryRecursiveNode.attributes = {
            value: opToken.lexeme,
            token: opToken,
        }
        this.advance()
        parentNode.children.push(unaryRecursiveNode)
        this.unary(unaryRecursiveNode)
    }
```
here we only need to change unaryRecursive function because it is responsible for handling unary operator ! and -, with the above code we can passed the test case above.
