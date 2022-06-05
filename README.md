# Testing
class MainActivity : AppCompatActivity() {
    private var adapter: ListAdapter? = null
    lateinit var listView: ListView
    val br = MyBroadcastReceiver(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        adapter = ListAdapter(this)
        listView = findViewById(R.id.list_view)
        listView.adapter = adapter
        reloadListView()
        registerReceiver(br, IntentFilter("UPDATE_USER_LIST"))
    }

    fun reloadListView() {
        adapter?.updateData(parseJson(getAllUsers()))
    }

    fun getAllUsers(): String {
        var result = ""
        val url = URL("http://restapiexample.com/api/v1/users")
        var connection: HttpURLConnection? = null
        try {
            connection = url.openConnection() as HttpURLConnection
            connection.connect()
            val reader = BufferedReader(InputStreamReader(connection.inputStream, Charsets.UTF_8))
            var line: String
            while ((reader.readLine().also { line = it }) != null) {
                result += line
            }
        } catch (e: Exception) {
            // do nothing
        } finally {
            connection!!.disconnect()
        }
        return result
    }

    private fun parseJson(json: String): List<Triple<Long, String, String>> {
        val list = mutableListOf<Triple<Long, String, String>>()
        val jsonObject = JSONObject(json)
        val jsonArray = jsonObject.getJSONArray("data")
        for (i in 0 until jsonArray.length()) {
            val obj = jsonArray.getJSONObject(i)
            val id = obj.getLong("id")
            val name = obj.getString("name")
            val image = obj.getString("avatar_url")
            list.add(Triple(id, name, image))
        }
        return list
    }

    class ListAdapter(private val context: Context) : BaseAdapter() {
        var data: List<Triple<Long, String, String>>? = null
        fun updateData(users: List<Triple<Long, String, String>>?) {
            this.data = users
            notifyDataSetChanged()
        }
        override fun getCount(): Int = data?.size ?: 0
        override fun getItem(position: Int): Any = data!![position]
        override fun getItemId(position: Int): Long = data!![position].first
        override fun getView(position: Int, convertView: View?, parent: ViewGroup): View {
            val view: View = convertView ?: LayoutInflater.from(context)
                .inflate(R.layout.item_user, parent, false)
            val nameView = view.findViewById<TextView>(R.id.name)
            nameView.text = data?.get(position)?.second
            val imageView = view.findViewById<ImageView>(R.id.image)
            Task(data!![position].third, imageView).execute()
            return view
        }
    }

    class Task(val avatarUrl: String, val imageView: ImageView) : AsyncTask<Void, Void, Bitmap>() {
        override fun doInBackground(vararg params: Void?): Bitmap? {
            return getBitmapFromURL(avatarUrl)
        }
        override fun onPostExecute(result: Bitmap?) = imageView.setImageBitmap(result)
        fun getBitmapFromURL(src: String?): Bitmap? {
            return try {
                val url = URL(src)
                val connection = url.openConnection() as HttpURLConnection
                connection.doInput = true
                connection.connect()
                BitmapFactory.decodeStream(connection.inputStream)
            } catch (e: Exception) {
                null
            }
        }
    }

    class MyBroadcastReceiver(private val activity: MainActivity) : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            activity.reloadListView()
        }
    }
}
