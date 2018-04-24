So my co-worker at this job challenged me to write a parser for the simple node layout in the question.  I did it combinator style in Perl, without using any libraries, then I did it in scala, using the stock parser combinator library there.

I'm impressed by the scala parser library, it's nice and powerful but requires less wonkery than haskell.

The node layout is: [A,[B,[C]],[D]]

So onto the perl solution:
First up, the comedy solution:
```perl
      my $str = '[A,[B,[C]],[D]]';
      $str =~ s/(\w+)/'$1'/g;
      say Dumper eval $str;
```

Hey, it works and it's readable.  Mostly readable.

Now, the perl combinator version:
```perl
    my $str = '[A,[B,[C]],[D]]';

    say Dumper parser()->() ;

    #Our structure must start with an array
    sub parser { return array() }

    #An array is a '[', followed by some elements, finished with a ']'
    sub array { sub { AND(st('['), elements(), element(),  st(']'))->() } }

    #Each array can be elements, separated by commas
    sub elements { sub { MANY( AND( element(), st(',') ) )->() } }

    #One element is either a sub-array, or a character
    sub element { sub { OR( array(), char() )->() } }


    #These are the support functions, in a normal parser, they would be part of the library
    #So I claim the real program is only the lines above

    #These functions are the equivalent of a "object factory".  They are a function factory.
    #They don't parse a string, they build a parser, that parses a string.
    #That is why we call parser()->() at the top of the program.
    #parser() creates a parser, and ->() actually parses


    #Match a letter
    sub char {
      return sub {
        $str =~ s/^(\w)//;
        return $1 if $1;
        return undef;
      }
    }

    #Match a string
    sub st {
      my $search = shift;
      return sub {
        $str =~ s/^(\Q$search\E)//;
        return $1 if $1;
        return undef;
      }
    }

    #Returns the result of the first successful parser
    sub OR {
      my @parsers = @_;
      return sub {
        my $oldStr = $str;
        foreach my $p (@parsers) {
          my $res = $p->();
          if ($res) {
            return $res;
          }
        }
        $str = $oldStr;
        return undef;
      }
    }

    #Always succeeds, returns true on its first failure
    sub MANY {
      my $p = shift;
      return sub {
        my @ret;
        while(1) {
          my $res = $p->();
          if (! $res ) {
            return [@ret];
          }
          push @ret,$res;
        }
      }
    }

    #Returns false on first failure, and resets the input string
    sub AND {
      my @parsers = @_;
      return sub {
        my $oldStr = $str;
        my @ret;
        foreach my $p (@parsers) {
          my $res = $p->();
          if (! $res ) {
            $str = $oldStr;
            return undef;
          }
          push @ret,$res;
        }
        return [@ret];
      }
    }
```

And now the scala version

```scala
        /* Parse an example from a job interview.
        Example data: [A,[B,[C]],[D]]
        */

        import scala.util.parsing.combinator.RegexParsers

        class MyParser extends RegexParsers {
            def f = array                                                  /* Top level must always be an array of nodes */
            def array:Parser[Any] = "[" ~ elements.* ~ element.*  ~ "]"    /* An array is a list of elements, separated by commas */
            def elements = element ~ ","                                   /* The comma separated part of the lsit */
            def element =  chara | array                                   /* The last element has no comma after it */
            def chara = "[a-zA-Z]".r                                       /* The only allowed node names are a-z and A-Z */
        }

        /* Let's test this a bit */
        val p = new MyParser
        Console.println( p.parseAll(p.f, "[a]") )
        Console.println( p.parseAll(p.f, "[a,b]") )
        Console.println( p.parseAll(p.f, "[a,c]") )
        Console.println( p.parseAll(p.f, "[[a],c]") )
        Console.println( p.parseAll(p.f, "[a,[c]]") )
        Console.println( p.parseAll(p.f, "[[a],[c]]") )
        Console.println( p.parseAll(p.f, "[A,[B,[C]],[D]]") )

        /* Even better are the error messages.  You get a useful error message from the parser when it fails to parse, without have to do any extra work */
        Console.println( p.parseAll(p.f, "[A,[B,[C]],[D]") )
```
