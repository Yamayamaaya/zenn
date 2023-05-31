---
title: "おれはこれをしたぞ"
---

# 5/31

### 業務

#### 分割代入（別名で）

```JavaScript
const person = {
  name: 'Alice',
  age: 30,
  occupation: 'Engineer',
  country: 'USA'
};

// オブジェクトのプロパティの分割代入
const { name, age } = person;

console.log(name); // 出力: Alice
console.log(age); // 出力: 30
const { name: personName, age: personAge } = person;

console.log(personName); // 出力: Alice
console.log(personAge); // 出力: 30
```

分割代入については把握していたが、別名で代入できることは知らなかった。

<br/>
<br/>

#### 非同期処理について

なんとなくでしか理解しておらず、ちゃんと説明できなかったため。

> 非同期処理は、プログラミングやコンピュータサイエンスの分野でよく使われる用語です。非同期処理は、処理の開始と完了が同期していないことを意味します。つまり、ある処理が開始された後でも、その処理が完了する前に他の処理が実行されることがあります。

> 一般的なプログラミングのモデルでは、処理は通常、順番に実行されます。つまり、1 つの処理が終了するまで次の処理は実行されません。しかし、非同期処理では、1 つの処理が開始された後に、別の処理が進行している間に他の処理も同時に実行されることがあります。非同期処理を利用することで、待ち時間が発生する処理や、ネットワークアクセスなどの入出力操作による遅延がある場合でも、他の処理を進行させながら処理を効率的に行うことができます。

-   代表的な方法三つ

    -   コールバック関数

        > 最も基本的な非同期処理の手法です。処理の完了時に呼び出される関数（コールバック関数）を指定します。処理が終了したら、その結果を引数としてコールバック関数が呼ばれます。

        ```JavaScript
        function fetchData(callback) {
        // 非同期のデータ取得処理
            setTimeout(function() {
                const data = { name: "John", age: 30 };
                callback(data); // 取得したデータをコールバック関数に渡す
            }, 2000); // 2秒後にデータ取得完了とする
        }

        // fetchData関数を呼び出し、コールバック関数を渡す
        fetchData(function(result) {
            console.log("取得したデータ:", result);
            // ここで取得したデータを使った追加の処理を行うことができる
        });
        ```

        データの取得が完了するまでの待ち時間中に他の処理を行うことができる。

    -   Promise
        > コールバックの代替手法として、プロミスが利用されることがあります。プロミスは、非同期処理の結果を表すオブジェクトであり、処理の完了またはエラーの結果を受け取るためのメソッドを提供します。プロミスは、処理の完了を待機するためのメソッド（例えば、then や catch）をチェーンすることができます。

    ```JavaScript
    function fetchData() {
        return new Promise(function(resolve, reject) {
            // 非同期のデータ取得処理
            setTimeout(function() {
            const data = { name: "John", age: 30 };
            resolve(data); // 取得したデータを解決(resolve)する
            }, 2000); // 2秒後にデータ取得完了とする
        });
    }

    // fetchData関数を呼び出し、Promiseオブジェクトを受け取る
    fetchData()
        .then(function(result) {
            console.log("取得したデータ:", result);
            // ここで取得したデータを使った追加の処理を行うことができる
        })
        .catch(function(error) {
            console.error("エラーが発生しました:", error);
    });
    ```

    -   async/await
        > async/await は、非同期処理を同期処理のように記述することができる構文です。async/await は、Promise をベースとしており、async 関数は Promise を返します。async 関数の中で await 演算子を使うことで、Promise の処理が完了するまで待機することができます。

    ```JavaScript
    function fetchData() {
        return new Promise(function(resolve, reject) {
            // 非同期のデータ取得処理
            setTimeout(function() {
            const data = { name: "John", age: 30 };
            resolve(data); // 取得したデータを解決(resolve)する
            }, 2000); // 2秒後にデータ取得完了とする
        });
    }

    async function getData() {
        try {
            const result = await fetchData(); // 非同期処理の完了を待機し、結果を受け取る
            console.log("取得したデータ:", result);
            // ここで取得したデータを使った追加の処理を行うことができる
        } catch (error) {
            console.error("エラーが発生しました:", error);
        }
    }

    // getData関数を呼び出す
    getData();
    ```

    async 自体は非同期処理し、async 関数の中では await を使って同期処理を行う。

    <br>
    <br>

#### DRY!!

先輩からレビューの際に指摘していただいた。知ってはいたが意識が足りなかった。

**_Don't Repeat Yourself !!!_**