# NIO BUFFER에 대해

## ※ Non-Direct, Direct Buffer

<br>

|         구분         | Non-Direct |            Direct           |
|:--------------------:|:----------:|:---------------------------:|
| 사용하는 메모리 공간 | JVM Heap   | OS Mem                      |
| 버퍼 생성 시간       | 빠름       | 느림                        |
| 버퍼의 크기          | 작다       | 크다(큰 데이터 처리 유리)   |
| 입출력 성능          | 낮다       | 높다(입출력 빈번할 때 유리) |

<br>

> 위 표를 바탕으로 NIO 버퍼를 사용할때 구분하여 사용 하면 되겠다.<br>
다만 Non-Direct 버퍼는 I/O를 위해 임시 Direct 버퍼를 생성하고 Non-Direct 버퍼에 존재하는<br>
내용을 임시 다이렉트 버퍼에 복사한다. 그리고 나서 다시 임시 Direct 버퍼를 사용해서 OS의 native I/O 기능을 수행 하기에 Direct 버퍼 보다는 I/O 성능이 낮다.

<br>

아래는 Non-Direct vs Direct 이다.

<br>

```java
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import java.util.EnumSet;

public class PerformanceExam {
	public static void main(String[] args) throws Exception {
		Path from = Paths.get("C:/TEST/1/1.txt");
		Path to1 = Paths.get("C:/TEST/2/2.txt");
		Path to2 = Paths.get("C:/TEST/3/3.txt");

		long size = Files.size(from);

		FileChannel fileChannel_from = FileChannel.open(from);
		FileChannel fileChannel_to1 = FileChannel.open(to1,
				EnumSet.of(StandardOpenOption.CREATE, StandardOpenOption.WRITE));
		FileChannel fileChannel_to2 = FileChannel.open(to2,
				EnumSet.of(StandardOpenOption.CREATE, StandardOpenOption.WRITE));

		ByteBuffer nonDirectBuffer = ByteBuffer.allocate((int) size);
		ByteBuffer directBuffer = ByteBuffer.allocateDirect((int) size);

		long start, end;

		/**
		 * P : Position, L : Limit C : Capacity
		 */
		start = System.nanoTime();
		for (int i = 0; i < 100; i++) {
			// 파일을 읽는다. P의 위치가 파일의 크기만큼 이동한다.
			fileChannel_from.read(nonDirectBuffer);
			// P와 L의 위치를 변경한다 P는 파일의 처음으로, L은 파일의 마지막으로 이동한다(읽기 준비).
			nonDirectBuffer.flip();
			// 파일을 쓴다.
			fileChannel_to1.write(nonDirectBuffer);
			nonDirectBuffer.clear(); // 재사용을 위해 버퍼를 비운다.
		}
		end = System.nanoTime();
		System.out.println("Non-Direct : " + (end - start) + " ns");

		fileChannel_from.position(0); // p의 위치를 0으로 설정한다.

		start = System.nanoTime();
		for (int i = 0; i < 100; i++) {
			// 파일을 읽는다. P의 위치가 파일의 크기만큼 이동한다.
			fileChannel_from.read(directBuffer);
			// P와 L의 위치를 변경한다 P는 파일의 처음으로, L은 파일의 마지막으로 이동한다(읽기 준비).
			directBuffer.flip();
			// 파일을 쓴다.
			fileChannel_to2.write(directBuffer);
			directBuffer.clear(); // 재사용을 위해 버퍼를 비운다.
		}
		end = System.nanoTime();
		System.out.println("Direct : " + (end - start) + " ns");

		fileChannel_from.close();
		fileChannel_to1.close();
		fileChannel_to2.close();
	}
}

-------------------------------------------

Non-Direct : 185238100 ns
Direct : 137290900 ns
```

<br>

> 결과를 보면 OS의 I/O를 사용하는 Direct버퍼가 더 빠른 결과를 보여준다.<br>Direct 버퍼는 데이터를 읽고 저장할 경우에만 OS의 native I/O를 수행한다. 만약 채널을 사용하지 않고<br>ByteBuffer의 get()/put() 메소드를 사용해서 버퍼의 데이터를 읽고, 저장한다면 이 작업은 내부적으로 JNI(Java Native Interface)를 호출 하기 때문에 오버 헤더가 추가 된다.<br>따라서 오히려 Non-Direct의 버퍼의 get()/put() 메소드 성능이 더 좋게 나올 수도 있다.

<br>

추가로 아래는 여러 버퍼 생성 방법이다.

```java
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.CharBuffer;
import java.nio.DoubleBuffer;
import java.nio.FloatBuffer;
import java.nio.IntBuffer;
import java.nio.LongBuffer;
import java.nio.ShortBuffer;

public class ManyDifferentBuffer {
	public static void main(String[] args) {
		/**
		 * JVM HEAP 메모리에 Non-Direct Buffer를 생성 allocate의 매개변수(개) 만큼의 값을 저장(버퍼 생성) 이외에
		 * warp() 메소드도 동일하다.
		 */
		ByteBuffer byteBuffer = ByteBuffer.allocate(100);
		CharBuffer charBuffer = CharBuffer.allocate(100);
		DoubleBuffer doubleBuffer = DoubleBuffer.allocate(100);
		FloatBuffer floatBuffer = FloatBuffer.allocate(100);
		IntBuffer intBuffer = IntBuffer.allocate(100);
		LongBuffer longBuffer = LongBuffer.allocate(100);
		ShortBuffer shortBuffer = ShortBuffer.allocate(100);

		/**
		 * OS 메모리에 Direct Buffer를 생성 형태 별로 Buffer 생성시 byteBuffer로 먼저 할당 후 asXXX 형태의 메소드
		 * 로 반환 받으면 된다.
		 */
		ByteBuffer directByteBuffer = ByteBuffer.allocateDirect(100);
		IntBuffer directIntBuffer = ByteBuffer.allocateDirect(100).asIntBuffer(); // 다른 형태도 마찬가지.

		/**
		 * 추가로 JVM은 Big endian으로 동작 하고 OS마다 Big Endian 인지 Little Endian 인지 구분 해야 한다. 다만
		 * OS와 바이트 해석 순서가 달라도 JVM에서 자동으로 처리해주므로 문제는 없다. 하지만 Direct Buffer일 경우 OS의 Native
		 * I/O를 사용하므로 명시적으로 OS의 해석 순서로 JVM의 해석순서를 맞춰주는 것이 성능에 도움이 된다.
		 */
		System.out.println("OS의 종류 : " + System.getProperty("os.name"));
		System.out.println("Native의 바이트 해석 순서 : " + ByteOrder.nativeOrder());

		// JVM의 해석 순서를 OS의 해석 순서로 맞추는 법.
		ByteBuffer directOsByteBuffer = ByteBuffer.allocateDirect(100).order(ByteOrder.nativeOrder());
	}
}
```