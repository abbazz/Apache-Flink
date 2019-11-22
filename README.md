# Apache-Flink
Twitter Hashtag Analysis using Apache Flink Platform

Step 1: Setup: Download and Start Flink

Install Apache Flink on your Windows/Linux/Ubuntu computer. To be able to run Flink, the only requirement is to have a working Java 8.x installation. Windows users, please take a look at the Flink on Windows guide which describes how to run Flink on Windows for local setups.
You can check the correct installation of Java by issuing the following command:

java -version

If you have Java 8, the output will look something like this:

java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)

1. Download a binary from the downloads page. You can pick any Scala variant you like. For certain features you may also have to download one of the pre-bundled Hadoop jars and place them into the /lib directory.
2. Go to the download directory.
3. Unpack the downloaded archive.

$ cd ~/Downloads        # Go to download directory
$ tar xzf flink-*.tgz   # Unpack the downloaded archive
$ cd flink-1.9.0

Step 2: Start a Local Flink Cluster

$ ./bin/start-cluster.sh  # Start Flink

Check the Dispatcher’s web frontend at http://localhost:8081 and make sure everything is up and running. The web frontend should report a single available TaskManager instance.

You can also verify that the system is running by checking the log files in the logs directory:

$ tail log/flink-*-standalonesession-*.log
INFO ... - Rest endpoint listening at localhost:8081
INFO ... - http://localhost:8081 was granted leadership ...
INFO ... - Web frontend listening at http://localhost:8081.
INFO ... - Starting RPC endpoint for StandaloneResourceManager at akka://flink/user/resourcemanager .
INFO ... - Starting RPC endpoint for StandaloneDispatcher at akka://flink/user/dispatcher .
INFO ... - ResourceManager akka.tcp://flink@localhost:6123/user/resourcemanager was granted leadership ...
INFO ... - Starting the SlotManager.
INFO ... - Dispatcher akka.tcp://flink@localhost:6123/user/dispatcher was granted leadership ...
INFO ... - Recovering all persisted jobs.
INFO ... - Registering TaskManager ... at ResourceManager

Step 3: Read the Code

Here's our pseudo code on Twitter Hashtag analysis. You can find the complete code on our Github Repository https://www.github.com/abbazz/Apache-Flink/ named code.java.


Scala
Java

package org.apache.flink.streaming.examples.twitter;

import java.io.Serializable;
import java.util.Date;
import java.util.Iterator;
import java.util.Properties;

public class TopTweet {

    public static void main(String[] args) throws Exception {
        // set up the execution environment
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        Properties props = new Properties();
        props.setProperty(TwitterSource.CONSUMER_KEY, "gQ8ZvjCR2aMMHNGlqys8G0kIS");
        props.setProperty(TwitterSource.CONSUMER_SECRET, "YG9NQZutuqveVM0aziai7u2tzt7GYat2Gv9yqGSwCrXfTKmv8f");
        props.setProperty(TwitterSource.TOKEN, "1598568512-3xiPKK0nw9RRkIcBMGMSlQSWsxeSrC4RqNJynHU");
        props.setProperty(TwitterSource.TOKEN_SECRET, "kQJRiuloYggbdKVewvcxNaT9m3lNimdb1VaDkPk3BrVhG");

        env.addSource(new TwitterSource(props))
            .flatMap(new ExtractHashTags())
            .keyBy(0)
            .timeWindow(Time.seconds(30))
            .sum(1)
            .filter(new FilterHashTags())
            .timeWindowAll(Time.seconds(30))
            .apply(new GetTopHashTag())
            .print();

        env.execute();
    }

    private static class TweetsCount implements Serializable {
        private static final long serialVersionUID = 1L;
        private Date windowStart;
        private Date windowEnd;
        private String hashTag;
        private int count;

        public TweetsCount(long windowStart, long windowEnd, String hashTag, int count) {
            this.windowStart = new Date(windowStart);
            this.windowEnd = new Date(windowEnd);
            this.hashTag = hashTag;
            this.count = count;
        }

        @Override
        public String toString() {
            return "TweetsCount{" +
                    "windowStart=" + windowStart +
                    ", windowEnd=" + windowEnd +
                    ", hashTag='" + hashTag + '\'' +
                    ", count=" + count +
                    '}';
        }
    }

    private static class ExtractHashTags implements FlatMapFunction<String, Tuple2<String, Integer>> {

        private static final ObjectMapper mapper = new ObjectMapper();

        @Override
        public void flatMap(String tweetJsonStr, Collector<Tuple2<String, Integer>> collector) throws Exception {
            JsonNode tweetJson = mapper.readTree(tweetJsonStr);
            JsonNode entities = tweetJson.get("entities");
            if (entities == null) return;

            JsonNode hashtags = entities.get("hashtags");
            if (hashtags == null) return;

            for (Iterator<JsonNode> iter = hashtags.getElements(); iter.hasNext();) {
                JsonNode node = iter.next();
                String hashtag = node.get("text").getTextValue();

                if (hashtag.matches("\\w+")) {
                    collector.collect(new Tuple2<>(hashtag, 1));
                }
            }
        }
    }

    private static class FilterHashTags implements FilterFunction<Tuple2<String, Integer>> {
        @Override
        public boolean filter(Tuple2<String, Integer> hashTag) throws Exception {
            return hashTag.f1 != 1;
        }
    }




Step 5: Further Steps

Run the above code in Eclipse or Netbeans Environment. Check the console to see the process flow of the system, and verify the output.
