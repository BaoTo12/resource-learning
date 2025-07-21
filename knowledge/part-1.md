### Nested Comment Design (Joe Celko)
![Nested Comment Design](./images/netest-comment.png)
![Nested Comment Design](./images/netest-comment-2.png)

__Nested Set Model__: A Nested Set Model or called **Modified Preorder Tree Traversal (MPTT)** is a way to represent hierarchical data in a relational database using a specific structure that ==includes left and right values for each node==, allowing for efficient querying of parent-child relationships. The way it traverses like pre-order traversal of Tree.

Traversal order is from 1 to 22
Formula for looking a child comment, for instance I wanna look for all child comments of comment 1.1.1 so 
>__The formula goes like this__: All comments that have index > 3 && index < 8. 


