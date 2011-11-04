!SLIDE
.notes What does this code return?
    @@@ ruby
    x = 1
    y = 2

    x + y

    # => ?

!SLIDE
.notes It returns 3.
    @@@ ruby
    x = 1
    y = 2

    x + y

    # => 3

!SLIDE
.notes What does this code return?
    @@@ ruby
    x <- 1
    y <- 2

    x + y

    # => ?

!SLIDE
.notes Well actually, I mean THIS code. What does this code return? Is it even valid?
    @@@ ruby
    Sequence.run do
      x <- 1
      y <- 2

      x + y
    end

    # => ?

!SLIDE
.notes It returns 3. How does it do that?
    @@@ ruby
    Sequence.run do
      x <- 1
      y <- 2

      x + y
    end

    # => 3

!SLIDE
.notes This is a magic trick in three parts.

!SLIDE
.notes First, let's pretend we wrote it like this.
    @@@ ruby
    Sequence.instance_eval do
      bind 1 do |x|
        bind 2 do |y|
          x + y
        end
      end
    end

!SLIDE
.notes Second, let's implement that mysterious 'bind' method. As you can see, it just calls the block you give it with the object you give it as an argument.
    @@@ ruby
    module Sequence
      class << self
        def bind(obj, &fn)
          fn.call(obj)
        end
      end
    end

!SLIDE
.notes Now the imaginary, not-the-actual-code code does what we want.
    @@@ ruby
    Sequence.instance_eval do
      bind 1 do |x|
        bind 2 do |y|
          x + y
        end
      end
    end

    # => 3

!SLIDE
.notes So how do we get the original code to work? We just include 'Transformer' in Sequence. Easy!
    @@@ ruby
    module Sequence
      class << self
        include Transformer

        def bind(obj, &fn)
          fn.call(obj)
        end
      end
    end

!SLIDE
.notes Ha ha ha. So anyway. Transformer needs to implement the 'run' method, and somehow turn the first block of code into the second.

!SLIDE
.notes The 'run' method calls a method to transform the block and then instance_evals it in the original block's lexical scope (that's what the second argument to eval does). Alright, but where's the definition of 'transform'?
    @@@ ruby
    module Transformer
      def run(&block)
        eval(ruby_for(block),
          block.binding).call
      end

      def ruby_for(block)
        %{
          #{self.name}.instance_eval {
            #{transform(block)}
          }
        }
      end
    end

!SLIDE
.notes Here it is. Wait, what? Ruby2Ruby? Rewriter?? block.to_sexp???????
    @@@ ruby
    module Transformer
      def transform(block)
        Ruby2Ruby.new.process(
          Rewriter.new.process(block.to_sexp))
      end
    end

!SLIDE
.notes Let me introduce my friends.

!SLIDE
.notes First up, Sourcify.

# gem install sourcify #

!SLIDE
.notes Sourcify provides many useful methods, but the one we're interested in is block.to_sexp. It parses a block's source and returns an abstract syntax tree.
    @@@ ruby
    proc { x + y }.to_sexp

!SLIDE
.notes That line returns this, which is called an S-expression.
    @@@ ruby
    s(:iter,
      s(:call, nil, :proc, s(:arglist)),
      nil,
      s(:call,
        s(:call, nil, :x, s(:arglist)),
        :+,
        s(:arglist,
          s(:call, nil, :y, s(:arglist)))))

!SLIDE
.notes Let's remove the s-es and commas though. Anyway, now that we have a way of getting at the syntax of a block, we can transform it.
    @@@ ruby
    (:iter
      (:call nil :proc (:arglist))
      nil
      (:call
        (:call nil :x (:arglist))
        :+
        (:arglist
          (:call nil :y (:arglist)))))

!SLIDE
.notes Here's what you get when you call .to_sexp on our code block.
    @@@ ruby
    (:block
      (:call
        (:call nil :x (:arglist))
        :<
        (:arglist
          (:call (:lit 1) :-@ (:arglist))))
      (:call
        (:call nil :y (:arglist))
        :<
        (:arglist
          (:call (:lit 2) :-@ (:arglist))))
      (:call
        (:call nil :x (:arglist))
        :+
        (:arglist
          (:call nil :y (:arglist)))))

!SLIDE
.notes And now you can see my trick. `x <- 1` is actually `x < (-1)`.
    @@@ ruby
    x <- 1
    # is actually
    x < (-1)

    (:call nil :x (:arglist))
    :<
    (:arglist
      (:call (:lit 1) :-@ (:arglist))))

!SLIDE
.notes So. In order to transform this into this...
    @@@ ruby
    Sequence.run do
      x <- 1
      y <- 2

      x + y
    end

    Sequence.instance_eval do
      bind 1 do |x|
        bind 2 do |y|
          x + y
        end
      end
    end

!SLIDE tiny-code
.notes ...we do this.
    @@@ ruby
    class Rewriter
      def process(exp)
        if exp[3].is_a?(Sexp) and exp[3][0] == :block
          iter, call, nil_val, block = exp.shift, exp.shift, exp.shift, exp.shift
          s(iter, call, nil_val, *rewrite_assignments(block[1..-1]))
        else
          exp
        end
      end

      def rewrite_assignments exp
        return [] if exp.empty?

        head = exp.shift


        if head[0] == :call and head[1] and head[1][0] == :call and head[2] == :< and head[3][0] == :arglist and head[3][1][2] == :-@
          var_name = head[1][2]
          expression = head[3][1][1]

          body = rewrite_assignments(exp)

          if body.first.is_a? Symbol
            body = [s(*body)]
          end

          [s(:iter,
            s(:call, nil, :bind, s(:arglist, expression)),
            s(:lasgn, var_name),
            *body)]
        elsif exp.empty?
          [head]
        else
          [s(:iter,
            s(:call, nil, :bind_const, s(:arglist, head)),
            nil,
            *rewrite_assignments(exp))]
        end
      end
    end

!SLIDE
.notes Ha ha ha. No, seriously. Here's the output. OK, but how do we turn it back into Ruby?
    @@@ ruby
    Rewriter.new.process(block.to_sexp)

    (:iter
      (:call nil :proc (:arglist))
      nil
      (:iter
        (:call nil :bind (:arglist (:lit 1)))
        (:lasgn :x)
        (:iter
          (:call nil :bind (:arglist (:lit 2)))
          (:lasgn :y)
          (:call
            (:call nil :x (:arglist))
            :+
            (:arglist
              (:call nil :y (:arglist)))))))

!SLIDE
.notes If you install sourcify, you get Ruby2Ruby for free. (It's a dependency.)
# Ruby2Ruby #

!SLIDE small-code
.notes If we run Ruby2Ruby on that S-expression we got just now, we get this.
    @@@ ruby
    Ruby2Ruby.new.process(
      DoNotation::Rewriter.new.process( block.to_sexp))

    "proc { bind(1) { |x| bind(2) { |y| (x + y) } } }"

!SLIDE small-code
.notes So if we run 'ruby_for' on our block, we get exactly what we wanted.
    @@@ ruby
    Sequence.ruby_for(proc do
      x <- 1
      y <- 2

      x + y
    end)

    "Sequence.instance_eval { bind(1) { |x| bind(2) { |y| (x + y) } } }"
