const paginate = async (params, modal) => {
    let docs = modal;
    const pagination = params.options?.pagination
    const searchKeys = params.search?.keys
    const searchValue = params.search?.value
    const populate = params.options?.populate
    let page = 1, limit = 4, skip;
    let query = {}

    if (searchKeys && searchValue) {
        if (searchKeys.length > 1) {
            query['$or'] = []
            searchKeys.forEach(element => {
                query['$or'].push({ [element]: { $regex: new RegExp(searchValue, "i") } })
            });
        } else if (searchKeys.length) {
            query = { [searchKeys[0]]: { $regex: new RegExp(searchValue, "i") } }
        }

    }
    docs = docs.find(query)
    console.log("Query", query);

    if (pagination) {
        console.log("Pagination", params);
        page = params.options.page;
        limit = params.options.limit;
        skip = (page - 1) * limit
        docs = docs.skip(skip).limit(limit)
    }
    console.log("Page :- ",page);
    console.log("limit :-",limit);
    console.log("Params :-",params.options.page);
    skip = (page - 1) * limit
    docs = docs.skip(skip).limit(limit)

    if (populate && populate.length) {
        populate.forEach(element => {
            docs = docs.populate(element)
        })
    }

    docs = await docs
    return docs;
};

module.exports = paginate;


----------------------------------------------

{
    "options": {
        "popuate": [
            {
                "path": "comments",
                "select": []
            },
        ],
        "select": [],
        "pagination": true,
        "page": 1,
        "limit": 2,
        "order": {}
    },
    "query": {

    },
    "search": {
        "keys": [],
        "value": "",
    }
}
