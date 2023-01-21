# Google API extend sheets query and drive file upload

The Google Visualization API Query Language (GQL) is not directly supported by the Google Sheets API. For that we can combine GQL with Google Sheets API. 

### Load gapi
```
<script src="https://apis.google.com/js/api.js"></script>
```

### Extent Sheets Query Code
```
const start = async () => {
  await gapi.client.init({
    discoveryDocs: [
      "https://sheets.googleapis.com/$discovery/rest?version=v4",
    ],
  });

  gapi.client.setToken({
    access_token: "", // oauth2 token or when you init gapi clint then automatically get the token
  });

  
  // extend getQuery with sheet api here spreadsheetId, query and sheet are mandatory
  gapi.client.sheets.spreadsheets.values.getQuery = ({
    spreadsheetId,
    range,
    query,
    sheet,
  }) => {
    return new Promise(async (resolve, reject) => {
      if (spreadsheetId && sheet && query) {
        try {
          let data = await (
            await fetch(
              `https://docs.google.com/spreadsheets/d/${spreadsheetId}/gviz/tq?access_token=${
                gapi.client.getToken().access_token
              }&tq=${query}&sheet=${sheet}${range ? `&range=${range}` : ""}`
            )
          ).text();

          if (data.indexOf("google.visualization.Query.setResponse(") >= 0) {
            let response = JSON.parse(
              data.substr(data.indexOf("{")).slice(0, -2)
            );
            if (response.status == "error") {
              reject({
                ...response,
                status: 400,
              });
            } else {
              let values = response.table.rows.map((v) => {
                return v.c.map((v_) => {
                  return v_.v;
                });
              });

              resolve({
                status: 200,
                result: {
                  values,
                },
              });
            }
          } else {
            reject({
              ...JSON.parse(data.replace(")]}'\n", "")),
              status: 400,
            });
          }
        } catch (err) {
          reject({
            status: 401,
            result: {
              error: err,
            },
          });
        }
      } else {
        reject({
          status: 400,
          result: {
            error: {
              message: "Invalid JSON payload received",
            },
          },
        });
      }
    });
  };

  let history = 90 * 24 * 60 * 60 * 1000;
  let now = new Date().getTime();

  gapi.client.sheets.spreadsheets.values
    .getQuery({
      spreadsheetId: "", // spredsheet Id 
      sheet: "history", // sheet name
      range: "history!A1:F40", // range it is optional
      query: `select E, D, F, B, C, A where (${now} - A) <= ${history}`, // gql 
    })
    .then(function (response) {
      let values = response.result.values;
      // Do something with the values
    });
};

gapi.load("client", start);
```

### Extend Drive upload
```
const start = async () => {
  await gapi.client.init({
    discoveryDocs: ["https://sheets.googleapis.com/$discovery/rest?version=v4"],
  });

  gapi.client.setToken({
    access_token: "", // oauth2 token or when you init gapi clint then automatically get the token
  });

  // extend upload big file with gapi drive files api. which is a class!
  gapi.client.drive.files.upload = class {
    constructor(options) {
      this.file = options.file;
      this.token = gapi.client.getToken().access_token;
      this.metadata = options.metadata;
      this.onError = options.onError;
      this.onComplete = options.onComplete;
      this.onProgress = options.onProgress;

      this.xhr = new XMLHttpRequest();
      this.xhr.open("POST", this.url, true);
      this.xhr.setRequestHeader("Authorization", "Bearer " + this.token);
      this.xhr.setRequestHeader("Content-Type", "application/json");
      this.xhr.setRequestHeader("X-Upload-Content-Type", this.file.type);
      this.xhr.setRequestHeader("X-Upload-Content-Length", this.file.size);
      this.xhr.onload = this.onXhrLoad.bind(this);
      this.xhr.onerror = this.onXhrError.bind(this);
      // this.xhr.upload.onprogress = this.onXhrProgress.bind(this);
      this.xhr.send(JSON.stringify(this.metadata));
    }

    onXhrLoad() {
      if (this.xhr.status === 200) {
        this.url = this.xhr.getResponseHeader("Location");
        this.sendFile();
      } else {
        this?.onError(JSON.parse(this.xhr.response));
      }
    }

    onXhrError() {
      this?.onError(this.xhr.response);
    }

    onXhrProgress(event) {
      this?.onProgress(event);
    }

    sendFile() {
      let xhr = new XMLHttpRequest();
      xhr.open("PUT", this.url, true);
      xhr.setRequestHeader("Content-Type", this.file.type);
      xhr.onload = () => {
        if (xhr.status === 200) {
          this?.onComplete(JSON.parse(xhr.response));
        } else {
          this?.onError(JSON.parse(xhr.response));
        }
      };
      xhr.onerror = this.onError.bind(this);
      xhr.upload.onprogress = this.onProgress.bind(this);
      xhr.send(this.file);
    }
  };
};


new gapi.client.drive.files.upload({
  file: file,
  metadata: fileMetadata,
  onError: function(response) {
    console.error(response);
  },
  onComplete: function(response) {
    console.log(response)
    console.log(`File ID: ${response.id}`);
  },
  onProgress: function(event) {
    console.log(`Upload progress: ${event.loaded} of ${event.total}`);
  }
});

gapi.load("client", start);
```