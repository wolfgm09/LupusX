# LupusX
This is my own first code language 

import re

# Define Token Types for LupusX (including 'wolf')
KEYWORDS = {'if', 'else', 'print', 'while', 'wolf'}
OPERATORS = {'+', '-', '*', '/', '=', '==', '!=', '<', '>'}
PARENTHESES = {'(', ')'}
WHITESPACE = {' ', '\t', '\n'}

# Token Specification with Regular Expressions (including 'wolf')
token_specification = [
    ('NUMBER', r'\d+'),                   # Integer
    ('STRING', r'"[^"]*"'),               # Double-quoted string
    ('KEYWORD', r'\b(?:if|else|print|while|wolf)\b'),  # Keywords, including 'wolf'
    ('IDENTIFIER', r'\b[A-Za-z_][A-Za-z0-9_]*\b'),  # Variable names
    ('OPERATOR', r'[+\-*/=<>!]=?|==|!=|<|>'),  # Operators
    ('PARENTHESIS', r'[()]'),             # Parentheses
    ('WHITESPACE', r'[ \t\n]+'),          # Whitespace (ignored)
    ('MISMATCH', r'.'),                   # Any other character (error)
]

# Compile Regular Expressions
master_pattern = '|'.join(f'(?P<{pair[0]}>{pair[1]})' for pair in token_specification)

def lexer(code):
    """Tokenize the input code using regex"""
    line_num = 1
    for mo in re.finditer(master_pattern, code):
        kind = mo.lastgroup
        value = mo.group()
        if kind == 'WHITESPACE':
            continue  # Skip whitespace
        elif kind == 'MISMATCH':
            raise RuntimeError(f"Unexpected character {value!r} on line {line_num}")
        yield kind, value

# ASTNode and subclasses to represent different parts of the code for LupusX
class ASTNode:
    """Base class for all AST nodes."""
    pass

class Program(ASTNode):
    def __init__(self, statements):
        self.statements = statements

class IfStatement(ASTNode):
    def __init__(self, condition, true_block, false_block=None):
        self.condition = condition
        self.true_block = true_block
        self.false_block = false_block

class AssignmentStatement(ASTNode):
    def __init__(self, identifier, value):
        self.identifier = identifier
        self.value = value

class PrintStatement(ASTNode):
    def __init__(self, expression):
        self.expression = expression

class WolfStatement(ASTNode):  # New class for the 'wolf' keyword
    def __init__(self, action):
        self.action = action

class Expression(ASTNode):
    """Base class for expressions."""
    pass

class BinaryExpression(Expression):
    def __init__(self, left, operator, right):
        self.left = left
        self.operator = operator
        self.right = right

class LiteralExpression(Expression):
    def __init__(self, value):
        self.value = value

class IdentifierExpression(Expression):
    def __init__(self, identifier):
        self.identifier = identifier

# Parser Class to Convert Tokens into AST for LupusX
class LupusXParser:
    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0

    def parse(self):
        """Parse tokens into AST."""
        return self.program()

    def current_token(self):
        """Get the current token."""
        return self.tokens[self.pos] if self.pos < len(self.tokens) else None

    def eat(self, token_type):
        """Consume a token if it matches the expected type."""
        if self.current_token() and self.current_token()[0] == token_type:
            self.pos += 1
        else:
            raise SyntaxError(f"Expected {token_type}, got {self.current_token()}")

    def program(self):
        """Parse the program."""
        statements = []
        while self.pos < len(self.tokens):
            statements.append(self.statement())
        return Program(statements)

    def statement(self):
        """Parse a single statement."""
        current = self.current_token()[0]
        
        if current == 'KEYWORD' and self.current_token()[1] == 'if':
            return self.if_statement()
        elif current == 'KEYWORD' and self.current_token()[1] == 'print':
            return self.print_statement()
        elif current == 'KEYWORD' and self.current_token()[1] == 'wolf':  # Detect wolf
            return self.wolf_statement()
        elif current == 'IDENTIFIER':
            return self.assignment_statement()
        else:
            raise SyntaxError(f"Unexpected statement: {self.current_token()}")

    def if_statement(self):
        """Parse an if statement."""
        self.eat('KEYWORD')  # eat 'if'
        condition = self.expression()
        self.eat('PARENTHESIS')  # eat ')'
        true_block = self.statement()
        
        false_block = None
        if self.current_token() and self.current_token()[0] == 'KEYWORD' and self.current_token()[1] == 'else':
            self.eat('KEYWORD')  # eat 'else'
            false_block = self.statement()
        
        return IfStatement(condition, true_block, false_block)

    def assignment_statement(self):
        """Parse an assignment statement."""
        identifier = self.current_token()[1]
        self.eat('IDENTIFIER')
        self.eat('OPERATOR')  # eat '='
        value = self.expression()
        return AssignmentStatement(identifier, value)

    def print_statement(self):
        """Parse a print statement."""
        self.eat('KEYWORD')  # eat 'print'
        self.eat('PARENTHESIS')  # eat '('
        expression = self.expression()
        self.eat('PARENTHESIS')  # eat ')'
        return PrintStatement(expression)

    def wolf_statement(self):  # Parse the 'wolf' keyword as a statement
        """Parse a wolf statement."""
        self.eat('KEYWORD')  # eat 'wolf'
        action = self.expression()  # Action associated with 'wolf'
        return WolfStatement(action)

    def expression(self):
        """Parse expressions (terms and operators)."""
        term = self.term()
        while self.current_token() and self.current_token()[0] == 'OPERATOR' and self.current_token()[1] in ('+', '-'):
            operator = self.current_token()[1]
            self.eat('OPERATOR')
            term = BinaryExpression(term, operator, self.term())
        return term

    def term(self):
        """Parse terms (factors and operators)."""
        factor = self.factor()
        while self.current_token() and self.current_token()[0] == 'OPERATOR' and self.current_token()[1] in ('*', '/'):
            operator = self.current_token()[1]
            self.eat('OPERATOR')
            factor = BinaryExpression(factor, operator, self.factor())
        return factor

    def factor(self):
        """Parse a factor (number, identifier, or parenthesized expression)."""
        current = self.current_token()
        
        if current[0] == 'NUMBER':
            self.eat('NUMBER')
            return LiteralExpression(current[1])
        elif current[0] == 'IDENTIFIER':
            self.eat('IDENTIFIER')
            return IdentifierExpression(current[1])
        elif current[0] == 'PARENTHESIS' and current[1] == '(':
            self.eat('PARENTHESIS')
            expr = self.expression()
            self.eat('PARENTHESIS')
            return expr
        else:
            raise SyntaxError(f"Unexpected factor: {current}")

# Interpreter Class to Execute the AST for LupusX
class Interpreter:
    def __init__(self):
        self.variables = {}

    def visit(self, node):
        """Visit nodes in the AST and execute them."""
        if isinstance(node, Program):
            return self.visit_program(node)
        elif isinstance(node, IfStatement):
            return self.visit_if_statement(node)
        elif isinstance(node, AssignmentStatement):
            return self.visit_assignment_statement(node)
        elif isinstance(node, PrintStatement):
            return self.visit_print_statement(node)
        elif isinstance(node, WolfStatement):  # Handle the 'wolf' statement
            return self.visit_wolf_statement(node)
        elif isinstance(node, BinaryExpression):
            return self.visit_binary_expression(node)
        elif isinstance(node, LiteralExpression):
            return node.value
        elif isinstance(node, IdentifierExpression):
            return self.visit_identifier_expression(node)

    def visit_program(self, node):
        """Execute the program."""
        for statement in node.statements:
            self.visit(statement)

    def visit_if_statement(self, node):
        """Execute an if statement."""
        condition_value = self.visit(node.condition)
        if condition_value:
            self.visit(node.true_block)
        elif node.false_block:
            self.visit(node.false_block)

    def visit_assignment_statement(self, node):
        """Execute an assignment statement."""
        value = self.visit(node.value)
        self.variables[node.identifier] = value

    def visit_print_statement(self, node):
        """Execute a print statement."""
        print(self.visit(node.expression))

    def visit_wolf_statement(self, node):  # Execute 'wolf' actions
        print(f"Wolf action: {self.visit(node.action)}")

    def visit_binary_expression(self, node):
        """Evaluate binary expressions."""
        left_value = self.visit(node.left)
        right_value = self.visit(node.right)
        if node.operator == '+':
            return left_value + right_value
        elif node.operator == '-':
            return left_value - right_value
        elif node.operator == '*':
            return left_value * right_value
        elif node.operator == '/':
            return left_value / right_value

    def visit_identifier_expression(self, node):
        """Retrieve the value of an identifier."""
        return self.variables.get(node.identifier, None)

# Example usage with some test code, including 'wolf'
code = '''
x = 5
if x == 5:
    wolf "Run with the pack!"
else:
    print("x is not five!")
'''

# Tokenize, Parse, and Execute the Code for LupusX
tokens = list(lexer(code))
parser = LupusXParser(tokens)
ast = parser.parse()

interpreter = Interpreter()
interpreter.visit(ast)
![LupusX (2)](https://github.com/user-attachments/assets/dd56126f-abe5-4f8f-8471-3fa6de926af4)
