# Concepts (P0734R0) Implementation In Clang
Roadmap for implementation of Concepts in the Clang compiler.

## Roadmap
1. There are no longer variable and function concepts, and using VarTemplateDecl to represent concepts is unnecessarily complex and would require many workarounds to prevent said variables from actually being instantiated.
   Therefore - remove the support for them (currently there is a flag in TemplateDecl and a 'concept' storage specifier, and a bunch of diagnostics which are no longer relevant).
   
2. Add a new "ConceptTemplateDecl" AST node which represents a concept declaration.
    - inherits from TemplateDecl and not RedeclarableTemplateDecl, because concepts are not redeclarable.
    - stores an Expr * representing the constraint expression in the function's body.
    - has a FoldingSetVector of already "instantiated" concepts with specified arguments called "ConceptSpecialization" which is not a Decl but simply a "cache entry" if you will.
    
3. Add code to parse "ConceptTemplateDecl"s.

4. Add a new "ConceptSpecializationExpr" expression which is a concept name with template arguments, which stores a pointer to ConceptTemplateDecl and a ConceptSpecialization.
    - when a ConceptSpecializationExpr is created without dependent template arguments, it is actually resolved to an existing "ConceptSpecialization" or creates one if it is not yet present.
    - translate during constant evaluation and code generation to a bool.
  
5. Add code to parse "ConceptSpecializationExpr"s.

*  There was already a "RequiresClause" field (by means of TrailingObjects) added to TemplateParameterList.
   
*  There was already a "ConstrainedTemplateDeclInfo" struct which contains a TemplateParameterList and a ConstraintExpression added to TemplateDecl. 
   Keep that for later caching a combined constraint expression with both the requires clause, constraints arising from constrained template parameters (a.k.a template<Callable C>), and from trailing requires clauses on functions (a.k.a void foo() requires sizeof(int) == 4). 
   
6. Remove ConstrainedTemplateDeclInfo from TemplateDecl constructors and instead create one in the constructor if there are any associated constraints in the TemplateParameterList passed in.
   
7. Add ConstrainedTemplateDeclInfo to VarPartialSpecializationTemplateDecl and ClassPartialSpecializationTemplateDecl as well.
   
8. Add a function to ClassPartialSpecializationTemplateDecl, VarPartialSpecializationTemplateDecl and TemplateDecl called 
   "getAssociatedConstraints", which will return the total associated constraints, collected from all sources mentioned above (this method will not do any calculation/creation of "and" expressions - the expression should be created in the ctor of said classes if it requires calculation).
   
9. Enforce RequiresClause on template specialization in the function: 
    - CheckTemplateArgumentList which is ran before a primary template is instantiated with arguments.
    - ConvertDeducedTemplateArguments which is ran before partial specializations of all kinds are instantiated with arguments.
    - Both above functions will use a single template function which checks whether a bunch of arguments satisfy the constraints imposed by a 'templatedecl-like' object - it would basically instantiate the constraint expression returned by the object's getAssociatedConstraints() function.
    
11. Add code to Sema::getMoreSpecializedTemplate to regard constraint expressions.
  
12. Create a function which given an Expr * representing a constraint expression known to have not been satisfied, emits diagnostics as to why it wasn't (it would have special cases for BinaryOperators && and ||, as well as ConceptSpecializationExprs).
    Call this function in TemplateSpecCandidateSet::NoteCandidates when appropriate.
  
13. Add code to parse a trailing requires-clause in a function declaration.
   
14. Add a ConstraintExpression field to FunctionDecl, add code to TemplateDecl to take the constraint into account (and store it in the constraint stored in ConstrainedTemplateDeclInfo).
    - Go over all places where FunctionDecls are created and make sure constraints are passed in if needed.
  
15. Add name mangling for the added ConstraintExpression (which is now part of the signature).
 
16. Add ConstraintExpression checking for redeclarations of functions (consider whether or not the ConstraintExpression matches when determining if a function declaration is a redeclaration).
 
17. Prohibit virtual functions from being declared with constraints.
 
18. Change Sema::IsOverload to regard functions with matching signatures but different constraints as overloads.
    
19. Make sure functions whose constraints are not satisfied cannot be referenced. We can achieve this by ommitting them from Sema::LookupName when the LookupNameKind is not LookupRedeclarationWithLinkage.
    
20. Change addOverload and addTemplateOverload to disallow adding functions whose constraints are not met.
    
21. Change isBetter to regard a candidate whose constraints subsume the other to be better.

22. Add diagnostics to OverloadCandidateSet::NoteCandidates when appropriate.
 
23. Add a ConstraintExpression field to TypeTemplateParmDecl, TemplateTemplateParmDecl and NonTypeTemplateParmDecl, to represent constraints imposed by 'constrained template parameters' (e.g. things such as template<Callable C>).
    
24. Add a "calculateAssociatedConstraints" function to TemplateParameterList  which returns the requires clause constraint-expression ANDed with all constrained template parameter's cosntraint-expressions, make TemplateDecl, VarPartialSpecializationTemplateDecl and     ClassPartialSpecializationTemplateDecl use this function when calculating the associated constraints in TemplateDecl.
    
25. Add code to parse constrained template parameters and generate the imposed constraint-expression, storing them in the created 
    TemplateParameters.
  
26. Add a new RequiresExpr expression class (e.g. requires(T t) { t.foo(); }).

27. Add code to parse RequiresExpr expressions.

28. Add code to the diagnostics functions we'd previously implemented to introspect into requires expressions which weren't satisfied to further explain why they weren't.
