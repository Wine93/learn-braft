文件服务
===

```proto
message GetFileRequest {
    required int64 reader_id = 1;
    required string filename = 2;
    required int64 count = 3;
    required int64 offset = 4;
    optional bool read_partly = 5;
}

message GetFileResponse {
    // Data is in attachment
    required bool eof = 1;
    optional int64 read_size = 2;
}

service FileService {
    rpc get_file(GetFileRequest) returns (GetFileResponse);
}
```