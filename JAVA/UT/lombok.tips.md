# lombok
## lombok とは？
JavaのボイラープレートコードをシンプルにしてくれるJavaのライブラリです。
例えばJavaBeansコードを書く際に、本ライブラリを使用すればgetterメソッド・setterメソッドをコード上に直接書かなくて済みます。
またプログラムが手続き的ではなく、宣言的になり可読性が高まります。
（Lombokは内部的にAST変換を利用しています。）

* インストール方法はリンク先を確認してください。
    * https://qiita.com/yyoshikaw/items/32a96332cc12854ca7a3

## Junit 時のハマり所
### 自動生成された所はカバレッジを通すのは大変
 実装したコードで自動生成されたボイラープレートコードを利用していない場合、
テスト時にコードを実行する必要があります。

しかし、`@Data` アノテーションの場合 getter、setter 以外にも equals,toString,canEquals ,hashCode が生成されますが、特にequalsのカバレッジを通すのは難しいです。

むやみに`@Data`をつけるとテスト時に大変な思いをするので、lombok を使う場合
必要なボイラープレートコードのみ生成するようにしましょう。

### `@Data`アノテーションを自動でテストしてくれるクラス
`@Data`アノテーションに対してのみ自動でテストを通してくれます。  
＊ただし、配列のフィールドが存在するクラスではテストに失敗します。  
＊力技で実装しているのでテスト中に警告文がコンソールに出力されます。気になる方は使わないほうが良いです。

#### pom.xml に追加
```xml :pom
<dependencies>
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-core</artifactId>
		<version>2.9.1</version>
	</dependency>
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-databind</artifactId>
		<version>2.9.1</version>
	</dependency>
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-annotations</artifactId>
		<version>2.9.1</version>
	</dependency>
	<dependency>
		<groupId>backend-common</groupId>
		<artifactId>backend-common</artifactId>
		<version>0.0.1</version>
	</dependency>
</dependencies>
```

####  LombokTestUtil.class
```java
import static org.junit.Assert.*;

import org.meanbean.test.BeanTester;

import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtConstructor;
import javassist.CtMethod;
import javassist.CtNewConstructor;
import javassist.CtNewMethod;
import javassist.Modifier;
import javassist.NotFoundException;
import nl.jqno.equalsverifier.EqualsVerifier;
import nl.jqno.equalsverifier.Warning;

/**
 * lombokで自動生成されたコードのjunit
 *
 */
public class LombokTestUtil {
	/**
	 * "@Data"の対応
	 * @param clz
	 */
	public static void doData(Class<?> clz) {
		// Test getters, setters and #toString
		new BeanTester().testBean(clz);

		// Test #equals and #hashCode
		EqualsVerifier.forClass(clz)// .withRedefinedSuperclass()
				.suppress(Warning.STRICT_INHERITANCE, Warning.NONFINAL_FIELDS).verify();

		// Test #equals and #hashCode
		EqualsVerifier.forClass(clz).withRedefinedSuperclass()
				.suppress(Warning.STRICT_INHERITANCE, Warning.NONFINAL_FIELDS).verify();

		// Verify not equals with subclass (for code coverage with Lombok)
		try {
			assertFalse(clz.newInstance().equals(createSubClassInstance(clz.getName())));
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	/**
	 * 子クラスを生成
	 * @param superClassName
	 * @return
	 * @throws NotFoundException
	 * @throws CannotCompileException
	 * @throws InstantiationException
	 * @throws IllegalAccessException
	 */
	private static Object createSubClassInstance(String superClassName)
			throws NotFoundException, CannotCompileException, InstantiationException, IllegalAccessException {
		ClassPool pool = ClassPool.getDefault();

		// Create the class.
		CtClass subClass = pool.makeClass(superClassName + "Extended");
		final CtClass superClass = pool.get(superClassName);
		subClass.setSuperclass(superClass);
		subClass.setModifiers(Modifier.PUBLIC);

		// Add a constructor which will call super( ... );
		CtClass[] params = new CtClass[] {};
		final CtConstructor ctor = CtNewConstructor.make(params, null, CtNewConstructor.PASS_PARAMS, null, null,
				subClass);
		subClass.addConstructor(ctor);

		// Add a canEquals method
		final CtMethod ctmethod = CtNewMethod.make(
				"public boolean canEqual(Object o) { return o instanceof " + superClassName + "Extended; }", subClass);
		subClass.addMethod(ctmethod);

		return subClass.toClass().newInstance();
	}
}
```

#### 使い方
```java
	@Test
	public void lombokTest() {
		LombokTestUtil.doData([testTarget].class);
	}
```

