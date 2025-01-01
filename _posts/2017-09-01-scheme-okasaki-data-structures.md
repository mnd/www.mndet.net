---
layout: default
title: Scheme & Okasaki data structures
---
Decided to read Okasaki work on lazy data structures. He made fix for SML syntax to achieve readable code for lazy structures in strict language. Same changes for scheme you can achieve in next manner:

    (use-modules
     (srfi srfi-45) ; lazy primitives
     (ice-9 match) ; pattern matching
     (ice-9 textual-ports))
    
    (define (read-delay chr port)
       (match (get-char port)
        (#\$ `(= force ,(read port)))
        (c (unget-char port c) `(delay ,(read port)))))
    (read-hash-extend #\$ read-delay)

And thats all. To delay evaluation you can use `#$expr`, to match delayed structures you can write `(match delayed_expr [#$$match ...])` (note: hash-dollar-dollar). And now you can write code very similay to SML code in Okasaki work:

    (define stream #$(cons 1 #$(cons 2 #$(cons 3 #$'()))
    (define (take-stream n s)
      #$(match (cons n s)
          [(0 . _) '()]
          [(_ . #$$'()) '()]
          [(k . #$$(h . t)) (cons h (take (- k 1) t))]))
