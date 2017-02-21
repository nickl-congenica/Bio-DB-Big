# NAME

Bio::DB::Big - Interface to BigWig and BigBed files via libBigWig

# SYNOPSIS

    use Bio::DB::Big;
    use Bio::DB::Big::AutoSQL;
    
    # Setup CURL buffers
    Bio::DB::Big->init();

    my $bw = Bio::DB::Big->open('path/to/file.bw');
    # Generic: get the type
    if($bw->is_big_wig()) {
      print "We have a bigwig file\n";
    }

    # Generic: Get headers
    my $header = $bw->header();
    printf("Working with %d zoom levels", $header->{nLevels});
    # Generic: Get chromosomes (comes back as a hash {chrom => length})
    my $chroms = $bw->chroms();
    
    #Get stats, values and intervals
    if($bw->has_chrom('chr1')) {
      my $bins = 10;
      
      # uses the zoom levels and returns an array of 10 bins over chromsome positions 1-100
      my $stats = $bw->get_stats('chr1', 0, 100, $bins, 'mean');
      foreach my $s (@{$stats}) {
        printf("%f\n", $s);
      }

      # Go directly to the raw level and calc on that but ask for maximum value per bin this time
      my $full_stats = $bw->get_stats('chr1', 0, 100, $bins, 'max', 1);

      # Get a value for each base over chromsome positions 1 - 100. Values can be undef if not set
      my $values = $bw->get_values('chr1', 0, 100);

      # Get the real intervals where a value was assigned
      my $intervals = $bw->get_intervals('chr1', 0, 100);
      foreach my $i (@{$intervals}) {
        printf("%d - %d: %f\n", $i->{start}, $i->{end}, $i->{value})
      }
      
      # Or iterate which allows you to move through a file without loading everything into memory
      my $blocks_per_iter = 10;
      my $iter = $bw->get_intervals_iterator('chr1', 0, 100, $blocks_per_iter);
      while(my $intervals = $iter->next()) {
        foreach my $i (@{$intervals}) {
          printf("%d - %d: %f\n", $i->{start}, $i->{end}, $i->{value})
        }
      }
    }

    my $bb = Bio::DB::Big->open('http://genome.ucsc.edu/goldenPath/help/examples/bigBedExample.bb');
    if($bb->is_big_bed) {
      my $with_string = 1;
      # Optionally you do not retrieve the "string" if you don't want to potenitally saving memory
      my $entries = $bb->get_entries('chr21', 9000000, 10000000, $with_string);
      foreach my $e (@{$entries}) {
        printf("%d - %d: %s\n", $e->{start}, $e->{end}, $e->{string});
      }

      # Or you can use an iterator
      my $blocks_per_iter = 10;
      my $iter = $bb->get_entries_iterator('chr21', 0, $bb->chrom_length('chr21'), $with_string, $blocks_per_iter);
      while(my $entries = $iter->next()) {
        foreach my $e (@{$entries}) {
          printf("%d - %d: %s\n", $e->{start}, $e->{end}, $e->{string});
        }
      }
      
      # Finally you can request AutoSQL and parse if available
      if($bb->get_autosql()) {
        my $autosql = $bb->get_autosql();
        my $as = Bio::DB::Big::AutoSQL->new($autosql);
        if($as->has_field('name')) {
          printf("%s: The field 'name' is in position %d\n", $as->name(), $as->get_field('name')->position());
        }
        # Or just get all fields as an arrayref
        my $fields = $as->fields();
      }
    }

# DESCRIPTION

This library provides access to the BigWig and BigBed file formats designed by UCSC. However rather than use kent libraries this uses libBigWig from [https://github.com/dpryan79/libBigWig](https://github.com/dpryan79/libBigWig) as it provides an implementation that avoids exiting when errors happen. libBigWig provides access to BigWig summaries, values and intervals alongside providing access to BigBed entries.

This implementation is read-only. Patches to give it write ability are welcomed however at the time of writing libBigWig only supports writing to BigWigs.

In addition there are a number of AutoSQL parsing objects implemented in Perl to provide some rough parsing capability when handling AutoSQL attached to a BigBed file. These are experimental but seem to work on a wide range of example AutoSQL fields.

Should you wish to use the kent library please consult [Bio::DB::BigFile](https://metacpan.org/pod/Bio::DB::BigFile), which is a very complete set of bindings into kent.

# INSTALLATION

Installation requires the following libraries to be made available

- **libBigWig** - [https://github.com/dpryan79/libBigWig](https://github.com/dpryan79/libBigWig)
- **libcurl**

We assume that libcurl is installed to a central location and is a requirement for libBigWig (especially if you want to access remote files). libBigWig can be located via the following mechanisms:

- By providing `--libbigwig=/path/to/libbigwig` to `Build.PL`
- Setting an environment variable `LIBBIGWIG_DIR` to the correct path
- Setting the `--prefix` argument
- Installing from [Alien::LibBigWig](https://metacpan.org/pod/Alien::LibBigWig)
- Using `pkg-config` to find the location
- Installing libBigWig to a central location. We attempt `/usr, /usr/local, /usr/share, /opt/local`

`Build.PL` looks to see if we can find `BigWig.h` and `libBigWig.a` in one of the above locations resolved in the above order. If we cannot find the library then compilation will fail.

# COORDINATE SYSTEMS USED IN THIS LIBRARY

This code is based on UCSC formats. Therefore all coordinates reported are expressed in 0-based, half-open. This means that a genomic coordinate  displayed on UCSC or Ensembl e.g. `chr1:1-100` is represented as `chr1 0 100`. To convert from 0-based, half-open to 1-base, fully-closed add 1 to the start.

# CLASS METHODS

## Bio::DB::Big->init();

Initalises libBigWig. Essential to call **if** you are going to load remote files. Consider doing this once in a BEGIN block in your code.

## my $bf = Bio::DB::Big->open('/path/to/big.file');

Perl method that wraps two methods from [Bio::DB::Big::File](https://metacpan.org/pod/Bio::DB::Big::File). File type is sniffed using `test_big_wig()`. If true we open the file using `open_big_wig()`. If not we open using `open_big_bed()`. The caller can then use `is_big_wig()` or `is_big_bed()` to assert the type of file now available.

# WORKING WITH BIG FILES

See [Bio::DB::Big::File](https://metacpan.org/pod/Bio::DB::Big::File) for more information on the routines available.

# WORKING WITH AUTOSQL

See [Bio::DB::Big::AutoSQL](https://metacpan.org/pod/Bio::DB::Big::AutoSQL) for more information on routines available. Also see [Bio::DB::Big::File](https://metacpan.org/pod/Bio::DB::Big::File) for the method `get_autosql()`.

# EXCEPTIONS

This library will raise exceptions as and when errors occur. You can trap them using eval or equivalent methods. The following are the class of exceptions raised (identified by the exception's prefix)

- **Invalid operation**

    Tried to use a bigwig method on a bigbed file or vice-versa

- **Invalid type**

    Unknown summary type given for statistics generation

- **Invalid chromosome**

    The chromosome was not found in this file

- **Invalid range**

    The specified range was incorrect. Normally caused when start is greater than end or end is greater than the chromosome length

- **Fetch error**

    Could not retrieve the requested region

- **Parse error**

    Could not parse a record. Normally happens with AutoSQL work.

# SEE ALSO

[Bio::DB::BigFile](https://metacpan.org/pod/Bio::DB::BigFile)

# LICENSE

Copyright \[2015-2017\] EMBL-European Bioinformatics Institute

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
