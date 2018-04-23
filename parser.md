So my co-worker at this job challenged me to write a parser for the simple node layout in the question.  I did it combinator style in Perl, without using any libraries, then I did it in scala, using the stock parser combinator library there.

I'm impressed by the scala parser library, it's nice and powerful but requires less wonkery than haskell.

The node layout is: [A,[B,[C]],[D]]

So onto the perl solution:
First up, the comedy solution:

    my $str = '[A,[B,[C]],[D]]';

    say Dumper parse($str);

    sub parse {
      my $str = shift;
      $str =~ s/(\w+)/'$1'/g;
      return eval $str;
    }

Hey, it works and it's readable.

Now, the perl combinator version:

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
    #
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


