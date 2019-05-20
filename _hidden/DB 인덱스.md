# DB 인덱스

## 인덱스

인덱스는 create index문을 사용하여 생성된 데이터베이스 안에 고유한 structure이다. 자체 디스크 스페이스를 필요로 하며 인덱스된 테이블 데이터의 복사본을 저장한다. 이는 인덱스가 순수한 중복임을 의미한다. 인덱스는 테이블을 참조하는 새로운 자료구조이다. 인덱스는 자체 공간이 필요하며, 중복이고, 다른 저장소에 저장된 실제 정보를 가리킨다.

핵심 개념은 모든 엔트리가 잘 정의된 순서로 정렬되어있다는 것이다. 정렬된 데이터셋에서 데이터를 찾는것은 매우 쉽고 빠르다. 인덱스는 insert, delete, update가 일어날때마다 인덱스 순서를 업데이터시켜야한다. 단 이때 데이터 이동은 최소화하는게 좋다. 

위와 같은 요구사항을 만족시키기 위해 데이터베이스는 Doubly Linked List와 Search Tree를 결합한 자료구조를 사용한다. 이 2개의 자료구조는 데이터베이스 성능의 특성 대부분을 설명한다.

### The Index Leaf Nodes

* 배열을 이용해서 인덱스를 생성하는 경우, 데이터의 삽입이 매우 힘들어진다. => 이러한 문제를 해결하기 위해 Doubly Linked List를 사용. 덕분에 새로운 데이터를 삽입할 때 데이터 이동없이 삽입이 가능
* 인덱스의 leaf 노드들을 연결하기 doubly linked list를 사용함 각 리프 노드는 데이터베이스 블락 혹은 페이지에 저장됨. 블락과 페이지는 데이터베이스의 가장 작은 저장 단위임
* 인덱스 블락은 모두 같은 크기이며 일반적으로 몇 킬로바이트임. 
  * 데이터베이스는 가능한 각 블록의 공간을 최대한으로 사용하여 가능한 많은 인덱스 엔트리를 각 블록에 저장함
* 인덱스 순서는 2개의 레벨로 유지됨
  * 각 리프 노드내의 인덱스 엔트리
  * 리프노드 간의 doubly linked list를 사용

![그림](https://use-the-index-luke.com/static/fig01_01_index_leaf_nodes.en.MMHwYDFb.png)

* 각 인덱스 엔트리는 인덱스된 칼럼과 ROWID로 이루어진다.
* 인덱스와 달리 테이블 데이터는 힙 자료구조 저장되며 전혀 정렬되지 않음
  * 같은 테이블 블락에 저장된 로우끼리 어떠한 관계가 없음
  * 블락끼리 어떠한 커넥션도 없음

### B-TREE

* 데이터베이스는 데이터를 빨리 찾기 위해 B-TREE를 사용함

![그림](https://use-the-index-luke.com/static/fig01_02_tree_structure.en.BdEzalqw.png)

*  The doubly linked list establishes the logical order between the leaf nodes. 
* The root and branch nodes support quick searching among the leaf nodes.
* Each branch node entry corresponds to the biggest value in the respective leaf node.
* a branch layer is built up until all the leaf nodes are covered by a branch node.
* The structure is a *balanced search tree* because the tree depth is equal at every position
  *  the distance between root node and leaf nodes is the same everywhere.
* Once created, the database maintains the index automatically. It applies every `insert`, `delete` and `update` to the index and keeps the tree in balance, thus causing maintenance overhead for write operations. 

![그림](https://use-the-index-luke.com/static/fig01_03_tree_traversal.en.niC7Q5jq.png)

* The tree traversal starts at the root node on the left-hand side. Each entry is processed in ascending order until a value is greater than or equal to (>=) the search term (57).
* The B-tree enables the database to find a leaf node quickly.
* It works almost instantly—even on a huge data set. That is primarily because of the tree balance, which allows accessing all elements with the same number of steps, and secondly because of the logarithmic growth of the tree depth.

### Slow Index

* Despite the efficiency of the tree traversal, there are still cases where an index lookup doesn't work as fast as expected.
* The first ingredient for a slow index lookup is the leaf node chain. 
  * There are obviously two matching entries in the index. At least two entries are the same, to be more precise
  * The database *must* read the next leaf node to see if there are any more matching entries.
  * That means that an index lookup not only needs to perform the tree traversal, it also needs to follow the leaf node chain

![그림](https://use-the-index-luke.com/static/fig01_03_tree_traversal.en.niC7Q5jq.png)

* The second ingredient for a slow index lookup is accessing the table. 
  *  Even a single leaf node might contain many hits—often hundreds. The corresponding table data is usually scattered across many table blocks 
  * That means that there is an additional table access for each hit.
* An index lookup requires three steps
  * (1) the tree traversal
  * (2) following the leaf node chain
  * (3) fetching the table data. 
  * (2), (3) steps might need to access many blocks—they cause a slow index lookup.
* The Oracle database has three distinct operations that describe a basic index lookup
  * INDEX UNIQUE SCAN : performs the tree traversal only. The Oracle database uses this operation if a unique constraint ensures that the search criteria will match no more than one entry.
  * INDEX RANGE SCAN : performs the tree traversal *and* follows the leaf node chain to find all matching entries. This is the fall­back operation if multiple entries could possibly match the search criteria.
  * TABLE ACCESS BY INDEX ROWID : operation retrieves the row from the table. This operation is (often) performed for every matched record from a preceding index scan operation.

The important point is that an INDEX RANGE SCAN can potentially read a large part of an index. If there is one more table access for each row, the query can become slow even when using an index.

## 단일 인덱스

* 인덱스는 칼럼 + ROWID로 구성
* 단일 인덱스는 1개 칼럼으로 구성된 인덱스
* 인덱스 탐색은 "수직적 탐색 => 수평적 탐색"으로 진행 
* Index Range Scan : 인덱스의 범위를 스캔함
* Index Unique Scan : 하나의 인덱스를 찾음

## 결합 인덱스

* 2개 이상의 칼럼으로 구성된 인덱스
  * 조건 칼럼이 인덱스 선두에 오는 것이 처리량이 적다.
  * 단 where 조건이 =으로 구성된 경우는 크게 신경쓰지 않아도 된다.
* 모든 조건절마다 최적화된 인덱스들을 모두 생성할 수는 없으므로 조건절에 자주 사용되는 칼럼들이 선정 대상임
* 선정된 칼럼들 중 "=" 조건으로 자주 조회되는 칼럼을 앞쪽에 위치하는 것이 유리
* 인덱스 Only로 해결할 수 있는 경우 칼럼 추가를 고려

> 인덱스는 하나의 쿼리만 보고 설계하는것이 아니다. 해당 테이블에서 발생하는 다양한 액세스패턴을 보고 최소의 인덱스로 최대의 효과를 끌어낼 수 있게 설계되어야 함

## Random Access

* 만약 where 조건에 부서번호와 연봉이 있는데, 인덱스는 부서번호로만 이뤄진 경우
  * 일단 부서번호를 이용해서 인덱스에서 해당하는 칼럼을 찾는다. 하지만 인덱스만으론 연봉에 대한 조건을 처리할 수 없다.
  * 부서번호로 찾은 모든 칼럼에 대해 연봉 조건이 맞는지 찾는다 (Random Access) 이때는 실제 테이블의 데이터를 가지고 연봉 조건에 해당하는 로우를 찾느다.
  * Random Access를 최소화하기 위해 인덱스에 칼럼 추가를 고려할 수 있음 (위와 같은 경우 연봉을 인덱스에 추가)

## 인덱스를 활용하지 못하는 경우

* 묵시적 형 변환이 발생하는 경우
* 칼럼 가공을 하는 경우
  * MEM_NO + 10
  * TO_CHAR(CREATE_DT, 'YYYYMM')
* LIKE와 같은 문자 비교형 함수 사용시 묵시적 형 변환이 발생할 수 있으멩 주의
  * LIKE 숫자 타입 비교시 인덱스 활용 불가