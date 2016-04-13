##通用SimpleBaseAdapter
Android中经常使用Adapter，每次重写Adapter会造成大量的重复代码。因此可以写一个抽象类，在里面封装一些不变的代码。
###思路
####继承BaseAdapter
通常我们以继承BaseAdapter的方式来创建我们的Adapter。继承BaseAdapter必须重写如下方法：

	@Override
    public int getCount() {
        return 0;
    }

    @Override
    public Object getItem(int position) {
        return null;
    }

    @Override
    public long getItemId(int position) {
        return 0;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        return null;
    }

为了避免大量的重复代码，应该将这些方法封装到抽象类中。

####ListView，GridView等的常见优化手段
在使用ListView的时候，我们最常见的优化手段就是convertView和ViewHolder，这部分的代码相对固定；应该也将这些方法封装到抽象类中。

###实现
考虑封装BaseAdapter的getCount,getItem,getItemId方法，这些方法的实现基本都一样，变数不大。这里我们可以在抽象类的构造方法中传入一个List做为数据源，指定为泛型

	public abstract class SimpleBaseAdapter<T> extends BaseAdapter {

    protected Context context;
    protected List<T> data;

    public SimpleBaseAdapter(Context context, List<T> data) {
        this.context = context;
        this.data = data == null ? new ArrayList<T>() : data;
    }

    @Override
    public int getCount() {
        return data.size();
    }

    @Override
    public Object getItem(int position) {
        if (position >= data.size())
            return null;
        return data.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }
}

这部分代码基本可以写死。

还剩BaseAdapter中的getView方法没有重写。开发中经常有大量代码放在这个方法中，虽然根据业务不同代码不尽相同，但还有有一些共性的东西可以抽象出来。

这里先将convertView和ViewHolder的逻辑抽象出来。
先看ViewHolder。使用ViewHolder的目的是为了节省findViewById()消耗的时间；将所有View打包成一个Holder，只需一次索引资源Id，后面就可以复用这个Holder。因此，在我们的抽象类中应该有一个getResId的抽象方法让子类实现；在我们的ViewHolder中会调用这个getResId方法，来实例化我们的Holder类。

 	public class ViewHolder {
        private SparseArray<View> views = new 			SparseArray<View>();
        private View convertView;
		
        public ViewHolder(View convertView) {
            this.convertView = convertView;
        }

        @SuppressWarnings("unchecked")
        public <T extends View> T getView(int resId) {
            View v = views.get(resId);
            if (null == v) {
                v = convertView.findViewById(resId);
                views.put(resId, v);
            }
            return (T) v;
        }
    }
    
     /**
     * 该方法需要子类实现，需要返回item布局的resource id
     * 
     * @return
     */
    public abstract int getItemResource();
    
解决了ViewHolder之后，再看convertView。convertView的不变逻辑是

	if(null == convertView){
		...//1、inflate指定的xml
		   //2.setTag 将convertView和ViewHolder绑定
	}else{
		...//getTag 得到ViewHolder
	}

因此，可以得到

	@SuppressWarnings("unchecked")
    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder holder;
        if (null == convertView) {
            convertView = View.inflate(context, getItemResource(), null);
            holder = new ViewHolder(convertView);
            convertView.setTag(holder);
        } else {
            holder = (ViewHolder) convertView.getTag();
        }
        return getItemView(position, convertView, holder);
    }
    
这里我们再定义一个抽象方法，目的是为了让子类实现getView的不同逻辑代码

	public abstract View getItemView(int position, View convertView, ViewHolder holder);
	
此外，在抽象类中再加上几个常用的数据刷新的方法

	public void addAll(List<T> elem) {
        data.addAll(elem);
        notifyDataSetChanged();
    }
	
    public void remove(T elem) {
        data.remove(elem);
        notifyDataSetChanged();
    }

    public void remove(int index) {
        data.remove(index);
        notifyDataSetChanged();
    }

    public void replaceAll(List<T> elem) {
        data.clear();
        data.addAll(elem);
        notifyDataSetChanged();
    }


