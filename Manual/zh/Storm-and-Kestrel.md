# Storm 与 Kestrel

本文说明了如何使用 Storm 从 Kestrel 集群中消费数据。

## 前言

### Storm

本教程中使用了 [storm-kestrel][1] 项目和 [storm-starter][2] 项目中的例子。建议读者将这几个项目 clone 到本地，并动手运行其中的例子。

### Kestrel

本文假定读者可以如[此项目][3]所述在本地运行一个 Kestrel 集群。

## Kestrel 服务器与队列

Kestrel 服务中包含有一组消息队列。Kestrel 队列是一种非常简单的消息队列，可以运行于 JVM 上，并使用 memcache 协议（以及一些扩展）与客户端交互。详情可以参考 [storm-kestrel][1] 项目中的 [KestrelThriftClient][4] 类的实现。

每个队列均严格遵循先入先出的规则。为了提高服务性能，数据都是缓存在系统内存中的；不过，只有开头的 128MB 是保存在内存中的。在服务停止的时候，队列的状态会保存到一个日志文件中。

请参阅[此文][5]了解更多详细信息。

Kestrel 具有 * 快速 * 小巧 * 持久 * 可靠 等特点。

例如，Twitter 就使用 Kestrel 作为消息系统的核心环节，[此文][6]中介绍了相关信息。

** 向 Kestrel 中添加数据

首先，我们需要一个可以向 Kestrel 的队列添加数据的程序。下述方法使用了 [storm-kestrel][1] 项目中的 `KestrelClient` 的实现。该方法从一个包含 5 个句子的数组中随机选择一个句子添加到 Kestrel 的队列中。

```java
  private static void queueSentenceItems(KestrelClient kestrelClient, String queueName)
            throws ParseError, IOException {

        String[] sentences = new String[] {
                "the cow jumped over the moon",
                "an apple a day keeps the doctor away",
                "four score and seven years ago",
                "snow white and the seven dwarfs",
                "i am at two with nature"};

        Random _rand = new Random();

        for(int i=1; i<=10; i++){

            String sentence = sentences[_rand.nextInt(sentences.length)];

            String val = "ID " + i + " " + sentence;

            boolean queueSucess = kestrelClient.queue(queueName, val);

            System.out.println("queueSucess=" +queueSucess+ " [" + val +"]");
        }
    }
```

## 从 Kestrel 中移除数据

此方法从一个队列中取出一个数据，但并不把该数据从队列中删除：

```java
private static void dequeueItems(KestrelClient kestrelClient, String queueName) throws IOException, ParseError { for(int i=1; i<=12; i++){

        Item item = kestrelClient.dequeue(queueName);

        if(item==null){
            System.out.println("The queue (" + queueName + ") contains no items.");
        }
        else
        {
            byte[] data = item._data;

            String receivedVal = new String(data);

            System.out.println("receivedItem=" + receivedVal);
        }
    }
```

此方法会从队列中取出并移除数据：

```java
private static void dequeueAndRemoveItems(KestrelClient kestrelClient, String queueName)
throws IOException, ParseError
     {
        for(int i=1; i<=12; i++){

            Item item = kestrelClient.dequeue(queueName);


            if(item==null){
                System.out.println("The queue (" + queueName + ") contains no items.");
            }
            else
            {
                int itemID = item._id;


                byte[] data = item._data;

                String receivedVal = new String(data);

                kestrelClient.ack(queueName, itemID);

                System.out.println("receivedItem=" + receivedVal);
            }
        }
}
```

## 向 Kestrel 中连续添加数据

下面的程序可以向本地 Kestrel 服务的一个 **sentence_queue** 队列中连续添加句子，这也是我们的最后一个程序。

可以在命令行窗口中输入一个右中括号 `]` 并回车来停止程序。

```java
import java.io.IOException;
import java.io.InputStream;
import java.util.Random;

import backtype.storm.spout.KestrelClient;
import backtype.storm.spout.KestrelClient.Item;
import backtype.storm.spout.KestrelClient.ParseError;

public class AddSentenceItemsToKestrel {

    /**
     * @param args
     */
    public static void main(String[] args) {

        InputStream is = System.in;

        char closing_bracket = ']';

        int val = closing_bracket;

        boolean aux = true;

        try {

            KestrelClient kestrelClient = null;
            String queueName = "sentence_queue";

            while(aux){

                kestrelClient = new KestrelClient("localhost",22133);

                queueSentenceItems(kestrelClient, queueName);

                kestrelClient.close();

                Thread.sleep(1000);

                if(is.available()>0){
                 if(val==is.read())
                     aux=false;
                }
            }
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        catch (ParseError e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }

        System.out.println("end");

    }
}
```

## 使用 KestrelSpout

下面的拓扑使用 `KestrelSpout` 从一个 Kestrel 队列中读取句子，并将句子分割成若干个单词（Bolt：SplitSentence），然后输出每个单词出现的次数（Bolt：WordCount）。数据处理的细节可以参考[消息的可靠性保证][6]一文。

```java
TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("sentences", new KestrelSpout("localhost",22133,"sentence_queue",new StringScheme()));
builder.setBolt("split", new SplitSentence(), 10)
            .shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 20)
        .fieldsGrouping("split", new Fields("word"));
```

## 运行

首先，以生产模式或者开发者模式启动你的本地 Kestrel 服务。

然后，等待大约 5 秒钟以防出现网络连接异常。

现在可以运行向队列中添加数据的程序，并启动 Storm 拓扑。程序启动的顺序并不重要。

如果你以 TOPOLOGY_DEBUG 模式运行拓扑你会观察到拓扑中 tuple 发送的细节信息。


[1]: https://github.com/nathanmarz/storm-kestrel
[2]: http://github.com/apache/storm/blob/master/examples/storm-starter
[3]: https://github.com/nathanmarz/storm-kestrel
[4]: https://github.com/nathanmarz/storm-kestrel/blob/master/src/jvm/backtype/storm/spout/KestrelThriftClient.java
[5]: https://github.com/nathanmarz/kestrel/blob/master/docs/guide.md
[6]: https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Guaranteeing-Message-Processing.md

