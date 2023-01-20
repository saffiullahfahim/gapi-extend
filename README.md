# Google Sheets API extend Query

The Google Visualization API Query Language (GQL) is not directly supported by the Google Sheets API. For that we can combine GQL with Google Sheets API. 

### Load gapi
```
<script src="https://apis.google.com/js/api.js"></script>
```

### Extent Code
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