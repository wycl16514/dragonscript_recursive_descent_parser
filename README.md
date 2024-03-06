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
