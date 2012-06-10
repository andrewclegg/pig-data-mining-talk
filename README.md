Getting the files
-----------------

Download the Kaggle MSD challenge data files from here:

    http://www.kaggle.com/c/msdchallenge/data

Download the list of all EchoNest IDs to artists/titles from here (additional file 1):

http://labrosa.ee.columbia.edu/millionsong/pages/getting-dataset

Download the list of dodgy song/track mappings from here:

    http://labrosa.ee.columbia.edu/millionsong/blog/12-2-12-fixing-matching-errors

Push all these files to HDFS if you're working on a cluster.

Finally, download and install DataFu from:

    http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.linkedin.datafu%22

Just put this in your `$PIG_HOME/lib` directory.

### These aren't currently in use but may be of interest

Download the Taste Subset triplets from here:

    http://labrosa.ee.columbia.edu/millionsong/tasteprofile

and unzip them into `train_triplets.txt`.

Recompress these using a splittable compressor, e.g. bzip2 or lzo, if required. I'm using bzip2 as it's IO-efficient at the expense of CPU, as I'm using a pseudo-distributed cluster with 4 mappers but only 1 physical disk.

Preparation
-----------

    -- Experimental!

    set pig.exec.mapPartAgg true;

    -- Enumrate iterates through a bag and gives each element an index number
    
    define Enumerate datafu.pig.bags.Enumerate('1');
    
    -- Load the log of mismatched songs so we can exclude them
    
    bad_songs = load 'sid_mismatches.txt'
        using org.apache.pig.piggybank.storage.MyRegExLoader('ERROR: <(\\w+) ')
        as song:chararray;
    
    -- Load raw data, and filter out the wrongly-matched songs, plus instances
    -- where user listened to song only once (they probably add noise and cost)
    
    triplets = load 'kaggle_visible_evaluation_triplets.txt' using PigStorage('\t')
        as (user:chararray, song:chararray, plays:int);
    
    triplets_filtered = foreach (
        filter (cogroup triplets by song, bad_songs by song)
            by (IsEmpty(bad_songs.song) and Not(IsEmpty(triplets.song)))) {
                non_singletons = filter triplets by plays > 1;
                valid_pairs = foreach non_singletons generate user, song;
                generate flatten(valid_pairs) as (user, song);
    }
    
    -- Replace big string keys with int IDs for efficiency
    
    song_ids1 = load 'kaggle_songs.txt' using PigStorage(' ')
        as (song:chararray, song_id:int);
    
    users = load 'kaggle_users.txt' using PigStorage() as user:chararray;
    
    user_ids = foreach (group users all)
        generate flatten(Enumerate($1)) as (user:chararray, user_id:long);
    
    triplets_j1 = join triplets_filtered by user, user_ids by user using 'replicated';
    
    triplets_j2 = join triplets_j1 by song, song_ids1 by song using 'replicated';
    
    fans = foreach triplets_j2 generate song_id as song_id, user_id as user_id;

### TODO

I tried to use FirstTupleFromBag in the `song_pairs` group expression below (instead of MAX), but it died with:

    java.lang.ClassCastException: datafu.pig.bags.FirstTupleFromBag cannot be cast to org.apache.pig.Accumulator    

Is this a bug?

Which are the most similar songs?
---------------------------------

Working definition: Two songs are 'similar' if they share a fanbase.

Naive approach: count the number of people each pair of songs has in common, and normalize it by the total number of unique listeners they have.

Rather than doing a `distinct` on their total listeners, we can get a count of uniques by adding their individual listener counts, then subtracting their shared listener count.

This measure is called Jaccard similarity.

You can think of it as a bipartite graph between songs and people, where the similarity between any two songs is a function of the proportion of edges for those songs which lead to someone who likes both of them.

_Hat tip to Jacob Perkins (@thedatachef) whose blog on doing the same with unipartite graphs was helpful:_

`http://thedatachef.blogspot.co.uk/2011/05/structural-similarity-with-apache-pig.html`

    -- Count number of listeners for each song
    
    fans_counts = foreach (group fans by song_id) generate
        flatten(fans), COUNT(fans) as total_fans;
    
    -- Filter out songs with only a single fan, to reduce noise and
    -- processing time
    
    edges1 = filter fans_counts by total_fans > 1;
    
    -- Now we need a copy of this relation, to join it to itself
    
    edges2 = foreach edges1 generate *;
    
    -- Construct the songs<->users bigraph, filtering out reflexive similarities
    
    bigraph = filter (join edges1 by user_id, edges2 by user_id)
        by edges1::fans::song_id != edges2::fans::song_id;
    
    -- For each song pair, calculate similarity from
    -- shared fans (intersection) and unique fans (union)
    
    connections = group bigraph by (edges1::fans::song_id, edges2::fans::song_id);
    
    song_pairs = foreach connections {
        isect_size = COUNT(bigraph);
        fans1 = MAX(bigraph.edges1::total_fans);
        fans2 = MAX(bigraph.edges2::total_fans);
        union_size = fans1 + fans2 - isect_size;
        generate 
            (double)isect_size / (double)union_size as jacsim,
            flatten(group) as (song1_id, song2_id),
            fans1 as fans1, fans2 as fans2, isect_size as overlap;
    }
    
    -- MAX is a bit of a hack in the previous statement; we know that total_fans
    -- is the same for every instance of a given song, but Pig doesn't know that
    
    -- For a selection of famous songs, what are the most similar ones?
    -- Everything from this point on is for display purposes really,
    -- the hard work's been done
    
    tracks = load 'unique_tracks.txt'
        using org.apache.pig.piggybank.storage.MyRegExLoader('^.*<SEP>(.*)<SEP>(.*)<SEP>(.*)')
        as (song:chararray, artist:chararray, title:chararray);

    -- Get rid of one->many track->song mappings arbitrarily

    songs1 = foreach (group tracks by song) {
        first = TOP(1, 0, tracks);
        generate flatten(first)
            as (song, artist, title);
    }
    
    songs2 = foreach songs1 generate *;
    
    song_ids2 = foreach song_ids1 generate *;

    -- Get the top 100 most popular tunes that are still in our dataset

    surviving_ids = foreach (group song_pairs by (song1_id, fans1))
        generate flatten(group) as (song_id, fans);

    top100 = limit (order surviving_ids by fans desc) 100;

    -- Get the best match for each one

    candidates = join song_pairs by song1_id, top100 by song_id using 'replicated';

    best_hits = foreach (group candidates by song1_id) {
        best_hit = TOP(1, 0, candidates); -- 0 == jacsim
        generate flatten(best_hit);
    }

    best_hits_j1 = join song_ids1 by song_id, best_hits by song1_id using 'replicated';

    best_hits_j2 = join song_ids2 by song_id, best_hits_j1 by song2_id using 'replicated';

    best_hits_j3 = join songs1 by song, best_hits_j2 by song_ids1::song using 'replicated';

    best_hits_j4 = join songs2 by song, best_hits_j3 by song_ids2::song using 'replicated';

    top100_titles = foreach best_hits_j4 generate
        jacsim, songs1::song, songs1::artist, songs1::title,
        songs2::song, songs2::artist, songs2::title,
        fans1, fans2, overlap;
    
    dump top100_titles;

Notice that the relationship between users and songs is symmetrical. We could use the same approach to find users based on similar tastes, just by changing how we construct the bigraph.

Similarity != recommendation
----------------------------

As it stands, this isn't intended to constitute a practical recommender system, although it could provide an input into one.

Really it's just an example of doing similarity search in Pig.

There are several reasons why a recommender would need much more work than this, but some of the obvious ones are:

* The cold start problem

What you do about new songs, or new users, that haven't accrued any plays yet.

* Lack of additional data sources

What about albums, bands, songwriters, genres, producers, even locations...

* The time dimension

Evolution of a user's tastes, or a band's style, over time.

* Novelty

A recommender that only makes obvious suggestions is no use.

Second-order similarity
-----------------------

This is an easy enhancement that related to the novelty issue, but has been used in other fields as well including text mining.

Songs A and C may not have many listeners in common, but there may be a third track B which has considerable (separate) overlaps with A and B.

You can use two copies of the `song_pairs` relation, joined together, to look for cases like this.

### Example?

Approximate methods
-------------------

I called this a naive approach because past a certain point it'll start getting costly to scale.

For very large data sets, you really need a way of partitioning the search space in such a way that you can do a local search for nearest neighbours instead of a global search. That is, you only need to compare each item to likely candidates for high similarity. 



