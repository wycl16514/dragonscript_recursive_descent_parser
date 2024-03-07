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

unary →  primary | unary_recursive

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
        this.stmt()
        treeRoot.children = this.parseTrees
        return treeRoot
    }

    stmt = () => {
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

After adding the above code we can make sure our test case can pass.
