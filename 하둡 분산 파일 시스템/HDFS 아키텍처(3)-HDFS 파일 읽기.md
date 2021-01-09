## HDFS의 파일 읽기

### 파일 조회 요청

클라이언트는 입력 스트림 객체를 이용해 HDFS에 저장된 파일을 조회할 수 있다.

![파일 조회 요청 과정](/img/하둡_분산_파일_시스템/파일_조회_요청_과정.png "파일 조회 요청 과정")

1. 클라이너트는 DistributedFileSystem의 `open` 메소드를 호출해 스트림 객체 생성을 요청한다.

2. DistributedFileSystem은 FSDataInputStream 객체를 생성한다. 이때 FSDataInputStream은 DFSDataInputStream과 DFSInputStream을 차례대로 래핑한다. DistributedFileSystem은 마지막에 래핑이 되는 DFSInputStream을 생성하기 위해 DFSClient의 open 메소드를 호출한다.

3. DFSClient는 DFSInputStream을 생성한다. 이때 DFSInputStream은 네임노드의 `getBlockLocations` 메소드를 호출해 조회 대상 파일의 블록 위치 정보를 조회한다.

4. 네임노드는 조회 대상 파일의 **블록 위치 목록을 클라언트에 가까운 순으로 정렬**한다. 정렬이 완료되면 DFSInputStream에 정렬된 블록 위치 목록을 반환한다. DistributedSystem은 DFSClient로부터 전달받은 DFSInputStream을 이용해 FSDataInputStream으로 생성해서 클라이언트에게 반환한다.

### 블록 조회

![블록 조회 과정](/img/하둡_분산_파일_시스템/블록_조회_과정.png "블록 조회 과정")

1. 클라이언트는 입력 스트림 객체와 `read` 매소드를 호출해 스트림 조회를 요청한다.

2. DFSInputStream은 첫 번째 블록과 **가장 가까운 데이터노드를 조회**한 후, 해당 블록을 조회하기 위한 리더기를 생성한다. 클라이언트와 블록이 저장된 데이터노드가 같은 서버에 있다면 로컬 블록 리더기인 BlockReaderLocal을 생성하고, 데이터노드가 원격에 있을 경우에는 RemoteBlockReader를 생성한다.

3. DFSInputStream은 리더기의 read 메소드를 호출해 블록을 조회한다. BlockReaderLocal은 로컬 파일 시스템에 저장된 블록을 DFSInputStream에게 반환한다. 그리고 RemoteBlockReader는 원격에 있는 데이터노드에게 블록을 요청하며, 데이터노드의 DataXceiverServer가 블록을 DFSInputStream에게 반환한다.

4. DFSInputStream은 파일을 **모두 읽을 때까지 계속해서 블록을 조회**한다. 만약 DFSInputStream이 저장하고 있던 블록을 모두 읽었는데도 파일을 모두 읽지 못했다면 네임노드의 `getBlockLocations` 메소드를 호출해 **필요한 블록 위치 정보를 다시 요청한다.** 파일을 끊김 없이 연속적으로 읽기 때문에 클라이언트는 **스트리밍 데이터를 읽는 것처럼 처리할 수 있다.**